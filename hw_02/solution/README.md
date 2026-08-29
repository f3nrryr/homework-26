# ДЗ 2. Проектирование взаимодействия сервисов — решение

> Условие: [../Задание.md](../Задание.md) · Таблица протоколов: [../Таблица протоколов.md](../Таблица%20протоколов.md)  
> Схемы складывайте в [diagrams/](diagrams/).

## Выбранная система

Доставка еды

## 1. Декомпозиция

_8 сервисов с описанием ответственности._

1. **RestaurantsHandbooksService** — микросервис-справочник (read-модель каталога).  
   Ответственность: названия заведений, описания, позиции меню с ценами и фото. Предоставляет данные для отрисовки в мобильном приложении клиента. Высокая read-нагрузка, активно кэшируется.

2. **PartnersService** — сервис партнёров и runtime-канал ресторана.  
   Ответственность:
   - заявки ресторанов на подключение и модерация меню;
   - принятие / отклонение заказа рестораном (Partner App / POS) и выставление статуса `PREPARING`.  
   Выделен отдельно из-за существенно меньшей нагрузки и другого жизненного цикла данных по сравнению с RestaurantsHandbooksService (разделение CQRS).

3. **OrdersService** — микросервис заказов.  
   Ответственность: создание, изменение и чтение заказов, хранение статуса и истории, запуск и реакция на события саги.

4. **PricingService** — расчёт итоговой цены заказа.  
   Ответственность: сервисный сбор, стоимость доставки (с учётом погоды, спроса, расстояния), применение промокодов. Объединяет бывшие CalculatorService и PromoDiscountsService, потому что оба всегда вызываются вместе в одной точке расчёта цены.

5. **PaymentsService** — платежи и выплаты.  
   Ответственность: authorize (hold) средств, capture, возвраты, выплаты курьерам. Расчёт мотивационных надбавок выполняется фоновым воркером внутри сервиса (отдельный MotivationService признан избыточным).

6. **CouriersService** — микросервис курьеров.  
   Ответственность: предоставление курьеру списка доступных заявок, назначение и освобождение курьера, управление статусами.

7. **CourierGeoService** — сервис геолокации курьеров.  
   Ответственность: приём высокочастотных обновлений координат от курьера и трансляция клиенту для трекинга.  
   **Обоснование отделения от CouriersService**: принципиально другой профиль нагрузки (тысячи location-update в секунду против редких CRUD-операций), другие требования к latency и горизонтальному масштабированию.

8. **ChatsService** — live-чат по заказу (клиент, курьер, поддержка).  
   **NotificationsService** — отправка push-уведомлений и email-чеков.

## 2. Протоколы

1. **RestaurantsHandbooksService** — REST (через API Gateway).  
   Публичный read-only каталог. Запросы шаблонные, высокая частота, хорошо кэшируются. GraphQL избыточен — набор полей фиксирован. Данные нужны синхронно для отрисовки UI.

2. **OrdersService** — REST (публичный через Gateway) + gRPC (к PricingService) + Kafka.  
   REST — создание, получение статуса и отмена заказа клиентом.  
   gRPC к PricingService — внутренний low-latency вызов со строгим контрактом (классическая ниша gRPC по таблице протоколов).  
   Kafka — публикация доменных событий (`OrderCreated`, `OrderStatusChanged` и др.).

3. **PricingService** — gRPC (основной протокол).  
   Только внутренние вызовы от OrdersService. gRPC обеспечивает производительность, типизацию и версионирование контракта при расчёте цены.

4. **CouriersService** — REST + gRPC (к CourierGeoService) + Kafka.  
   gRPC используется для получения актуальных координат при поиске ближайшего свободного курьера.

5. **PaymentsService** — REST (через Gateway) + Kafka + REST к внешним платёжным провайдерам.  
   Внешние вызовы выполняются синхронно с Idempotency-Key, таймаутом и Circuit Breaker.

6. **PartnersService** — REST + Kafka.  
   Включает как онбординг и модерацию, так и accept/reject заказа рестораном. Нагрузка значительно ниже клиентской, REST достаточен.

7. **ChatsService** — WebSocket (основной канал) + REST (история сообщений и fallback при реконнекте).  
   WebSocket необходим для двустороннего live-чата. REST-эндпоинт истории нужен для восстановления контекста после разрыва соединения.

8. **CourierGeoService** — MQTT (uplink от курьера) + WebSocket/SSE (downlink клиенту) + gRPC (для CouriersService).  
   MQTT рекомендован таблицей протоколов для мобильных сетей (экономия батареи и трафика при нестабильном канале). Альтернатива — HTTP batch location updates. Downlink клиенту — WebSocket для живого трекинга.

9. **NotificationsService** — Kafka (consume) + асинхронные вызовы APNs/FCM/Email с retry, backoff и DLQ.

## 3. Схема взаимодействия

Схема взаимодействия сервисов с указанием протоколов приложена в папке [diagrams/](diagrams/).

На схеме:
- сплошные стрелки — синхронные вызовы (REST / gRPC);
- пунктирные стрелки — асинхронные события Kafka;
- показаны API Gateway, Kafka, базы данных сервисов, Redis, внешние системы;
- явно отмечены рёбра Orders → Pricing (gRPC), Couriers → CourierGeo (gRPC), канал ресторана и WebSocket у клиента.

## 4. API-контракты

### 4.1. OrdersService

Базовый путь: `/api/v1/orders`

**Создание заказа**  
`POST /`  

Headers:
- `Authorization: Bearer <access_token>`
- `Idempotency-Key: <uuid>` (обязателен)

Тело запроса (клиент **не** передаёт `price` и `userId` — они берутся с сервера и из токена соответственно):

```json
{
  "restaurantId": "uuid",
  "items": [
    {
      "menuItemId": "uuid",
      "quantity": 2
    }
  ],
  "deliveryAddress": {
    "street": "Тверская ул., д. 1",
    "apartment": "45",
    "comment": "домофон 123"
  },
  "promoCode": "SUMMER10"
}
```

Ответ `201 Created`:

```json
{
  "orderId": "uuid",
  "status": "PENDING",
  "pricing": {
    "subtotal": 501.00,
    "deliveryFee": 99.00,
    "serviceFee": 25.00,
    "discount": 50.00,
    "total": 575.00
  },
  "createdAt": "2026-07-20T10:00:00Z"
}
```

Возможные ошибки: `400` (валидация), `401`, `402` (не удалось сделать hold), `404` (позиция или ресторан не найдены), `409` (конфликт идемпотентности или цена изменилась), `422`, `429`.

**Получение статуса заказа**  
`GET /{orderId}` — доступен только владельцу заказа (проверка по JWT).  
Поле `currentLocation` не возвращается (клиент получает координаты по отдельному WebSocket из CourierGeoService).

**Отмена заказа**  
`POST /{orderId}/cancel`  

Headers: `Idempotency-Key`  

Тело (опционально): `{"reason": "changed_mind"}`  

Бесплатная отмена возможна до перехода в статус `PREPARING`.

### 4.2. Контракт Kafka-события OrderCreated

```json
{
  "eventId": "uuid",
  "eventType": "OrderCreated",
  "eventVersion": 1,
  "occurredAt": "2026-07-20T10:00:00Z",
  "partitionKey": "orderId",
  "payload": {
    "orderId": "uuid",
    "restaurantId": "uuid",
    "userId": "uuid",
    "items": [],
    "pricing": {},
    "deliveryAddress": {}
  }
}
```

### 4.3. PartnersService — принятие / отклонение заказа рестораном

- `POST /api/v1/partners/orders/{orderId}/accept`
- `POST /api/v1/partners/orders/{orderId}/reject` (с указанием причины)

### 4.4. ChatsService

Протокол: WebSocket (`wss://`)  
Путь: `/ws/chat/{orderId}?role={client|courier|support}`  

Дополнительно REST: `GET /api/v1/chats/{orderId}/history` — история сообщений для реконнекта и аудита.

### 4.5. CourierGeoService

- Uplink от курьера: MQTT (или HTTP batch)  
- Downlink клиенту: WebSocket `/ws/order/tracking?orderId={uuid}`

## 5. Асинхронность

- Все критические изменения состояния публикуются через **Transactional Outbox** → Kafka.
- **Partition key = orderId** обеспечивает порядок событий одного заказа.
- Все консьюмеры сделаны **идемпотентными** (по `eventId` или комбинации `orderId + eventType`).
- Реализованы retry с exponential backoff и **DLQ** для сообщений, которые не удалось обработать.
- При пиковых нагрузках: rate limiting и backpressure на API Gateway, асинхронная обработка нотификаций и geo-обновлений.
- PaymentsService → платёжный провайдер: синхронный REST с Idempotency-Key и таймаутом. При таймауте запускается компенсация.
- NotificationsService: полностью асинхронный (чтение из Kafka → retry к APNs/FCM/Email).
- PartnersService: синхронный REST для accept/reject + публикация соответствующих событий в Kafka.

## 6. Паттерны

### 6.1. Saga (хореография) с подтверждением ресторана

Заказ затрагивает несколько независимых сервисов. Для сохранения целостности без распределённых транзакций используется Saga в стиле хореографии.

**Почему хореография, а не оркестрация (trade-off):**
- Плюсы: слабая связанность — каждый сервис знает только о шине событий (Kafka) и не вызывает другие сервисы напрямую.
- Минусы: состояние саги размазано, сложнее наблюдаемость и отладка, нет единой точки для глобальных таймаутов. Оркестрация дала бы централизованный контроль, но увеличила бы связанность и создала бы потенциальную точку отказа.

**Поток создания заказа:**

1. OrdersService принимает запрос, синхронно перепроверяет актуальные цены через PricingService (gRPC), сохраняет заказ в статусе `PENDING` и через Outbox публикует `OrderCreated`.
2. PartnersService получает событие. Ресторан через Partner App принимает или отклоняет заказ (таймаут 5–7 минут).  
   - `RestaurantAccepted` → статус `ACCEPTED_BY_RESTAURANT`  
   - `RestaurantRejected` / timeout → компенсация.
3. При Accepted PaymentsService выполняет **authorize (hold)** средств (не capture).  
   - Успех → `PaymentAuthorized`  
   - Ошибка → `PaymentFailed`.
4. CouriersService на `PaymentAuthorized` ищет ближайшего свободного курьера (координаты получает по gRPC из CourierGeoService).  
   - Находит → `CourierAssigned`  
   - Не находит / timeout → `CourierUnavailable`.
5. Только после `CourierAssigned` PaymentsService выполняет **capture**. OrdersService переводит заказ в `PREPARING` / `ACTIVE`.

**Компенсации** при любом отказе или таймауте:  
OrdersService публикует `OrderCompensationRequired` → PaymentsService делает release hold / refund, CouriersService освобождает курьера, PartnersService уведомляет ресторан.

### 6.2. Transactional Outbox

В сервисах-источниках событий (Orders, Payments, Couriers, Partners) используется таблица OutboxMessages. Запись доменных данных и события выполняется в одной ACID-транзакции. Фоновый воркер надёжно доставляет сообщения в Kafka.  

Паттерн даёт семантику **at-least-once**, поэтому все консьюмеры обязаны быть идемпотентными.

### 6.3. CQRS для справочников

PartnersService отвечает за запись (Command): создание, обновление меню, модерацию, принятие заказов.  
RestaurantsHandbooksService отвечает за чтение (Query): быстрая выдача каталога клиентам.  

Синхронизация — асинхронно через Kafka (`MenuUpdated`).  

**Следствие eventual consistency**: клиент теоретически может увидеть устаревшую цену.  
**Как закрываем**: при создании заказа OrdersService синхронно перепроверяет цены через PricingService. При расхождении возвращается `409 Conflict` с актуальной ценой. Заказ никогда не создаётся по устаревшим данным.

### 6.4. Дополнительные паттерны

- **Idempotent Consumer** — обязателен для всех обработчиков Kafka-событий.
- **Circuit Breaker + retry with exponential backoff** — для вызовов внешних платёжных провайдеров и APNs/FCM.
- **API Gateway / BFF** — единая точка входа, аутентификация, rate limiting, маршрутизация.
