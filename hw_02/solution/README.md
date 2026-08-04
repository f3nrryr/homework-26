# ДЗ 2. Проектирование взаимодействия сервисов — решение

> Условие: [../Задание.md](../Задание.md) · Таблица протоколов: [../Таблица протоколов.md](../Таблица%20протоколов.md)
> Схемы складывайте в [diagrams/](diagrams/).

## Выбранная система

_Доставка еды_

## 1. Декомпозиция

_6–8 сервисов с описанием ответственности._  
1. _RestaurantsHandbooksService_ -- микросервис-справочник с позициями еды из заведений.  
Сервис предоставляет эндпоинты для отрисовки на фронтенде (в мобильном приложении типа Delivery club).  
Ответственность -- названия заведений, описания, позиции из них с ценами и фото позиций.  

2. _OrdersService_ -- микросервис заказов (оплаченных заявок на доставку еды клиенту).  
Ответственность -- принимать (создавать), изменять заказы, читать заказы (для просмотра статуса и истории заказов клиентом на мобилке).

3. _CouriersService_ -- микросервис курьеров.  
Ответственность -- предоставлять курьеру список заявок, на определенном расстоянии (по бизнес-логике), которые он может забрать, с отрисовкой цен, целевого адреса.  
 
4. _ChatsService_ -- на случай инцидентов и пожеланий. Связь с технической поддержкой. Либо самим курьером. 3 актора -- клиент, саппорт и курьер могут друг другу писать. 
 
5. _NotificationsService_ -- микросервис отправки push-уведомлений, а также email-чеков по заказам.  

6. _PartnersService_ -- отдельный микросервис, где рестораны, заведения смогут оставлять заявки на появление в сервисе доставки. А также редактировать инфо о своих позициях (тоже будут проверять модераторы после заявок). Модераторы будут своевременно их рассматривать и в случае соблюдения бизнес-правил отображать в выдаче клиенту.  Выделяю в отдельный сервис из соображений наличия разной нагрузки на сервис RestaurantsHandbooks и на этот. Т.е. на первый сервис нагрузка будет больше. Ради масштабирования делим.  

7. _PaymentService_ -- оплата клиентом заказа, возврат клиенту денег по бизнес-логике, выплата курьеру оплаты.  

8. _MotivationService_ -- расчет KPI-надбавок курьерам и техподдержке (?).  

9. _CalculatorService_ -- микросервис, считающий сервисный сбор и цену на доставку в зависимости от условий (погода, количество заказов, количество курьеров).  

10. _PromoDiscountsService_ -- микросервис скидок по промо-кодам.  
  
11. _CourierGeoService_ -- сервис геолокации курьера для отрисовки пользователю на мобилке.  
  
## 2. Протоколы

_REST / gRPC / GraphQL / events — где и почему._  
  
1. _RestaurantsHandbooks_ -- REST.  
_Обоснование_:
a) постоянных апдейтов (как лайв-чат) не требуется. по запросу от клиентского приложения.  
b) всегда запрашиваем данные шаблонно (одни и те же поля для набора сущностей, можно делать REST-endpoint с JSON). GraphQL бы потребовал более сложной логики бэка, а как таковой гибкости нам тут не особо нужно.  
c) можно кэшировать на N времени или принудительно сбрасывать кэш, поэтому не ожидаем сверх-нагрузки, с которой REST не справится.  
~~d) у REST нет нюансов с авторизацией, как в gRPC. а мы будем вызывать API снаружи, так что авторизация нужна. ВЫЗЫВАЕМ из-за ШЛЮЗА~~  
e) данные мобилке нужны СЕЙЧАС, СИНХРОННО.  
  
2. _OrdersService_ -- REST -- со своей базой данных. Отброс события в топик Kafka -- для вычитки другими связанными сервисами (нотификации, курьеры, партнеры, платежи, мотивация).    
_Обоснование_:  
a) Вызов напрямую из API Gateway.  
b) Справочники можно кэшировать (статус заказа и т.п).  
c) всегда запрашиваем данные шаблонно (одни и те же поля для набора сущностей, можно делать REST-endpoint с JSON). GraphQL бы потребовал более сложной логики бэка, а как таковой гибкости нам тут не особо нужно.  
  
3. _CouriersService_ -- REST -- со своей базой данных. Отброс события в топик Kafka -- для вычитки другими связанными сервисами (Orders, Notifications, Chats, Payments, Users, Calculator, CourierGeo).  
_Обоснование_:  
a) Вызов напрямую из API Gateway.  
b) Справочники можно кэшировать.  
c) всегда запрашиваем данные шаблонно (одни и те же поля для набора сущностей, можно делать REST-endpoint с JSON). GraphQL бы потребовал более сложной логики бэка, а как таковой гибкости нам тут не особо нужно.  
   
4. _ChatsService_ -- WebSocket.  
_Обоснование_:  
a) Необходим лайв-чат. Двухсторонний.  
  
5. _NotificationsService_ -- отправляет пуши через сторонние API Google/Apple. Или Email. Т.е. это интеграция с нашей стороны с кем-то внешним.  
Таким образом выбираю MQ + вычитку воркером (async) и отправку по API партнера.    
_Обоснование_:  
Ожидается много запросов, но не очень большими данными (стандартные шаблоны писем на почту либо краткие пуши).  
  
6. _PartnersService_ -- REST -- со своей базой данных. Отброс события в топик Kafka -- для вычитки другими связанными сервисами (Orders, Restaurants, Payments, Promo).  
_Обоснование_:  
a) Ожидается не столь много трафика, т.к. самих ресторанов и частота обновления меньше, чем юзеров-клиентов.  
b) Справочники можно кэшировать.  
c) Вызов из API Gateway.  
d) Данные шаблонные.  
  
7. _PaymentsService_ -- REST -- со своей базой. Интеграция с платежными системами. Отброс события в топик Kafka -- для вычитки другими связанными сервисами (Orders, Motivation, Restaurants).  
_Обоснование_:  
a) Кэш справочников.  
b) Ожидается много запросов, но не очень большими данными (стандартные шаблоны писем на почту либо краткие пуши). Поэтому не вижу смысла с ходу в gRPC из-за его усложнений.  
  
8. _MotivationService_ -- можно сохранять данные, но KPI пересчитывать гораздо реже их получения. REST.  
a) Кэш справочников.  
b) Ожидается много запросов, но не очень большими данными.  
c) Строгий шаблон данных.  
  
9. _CalculatorService_ -- REST.  
a) Кэш справочников.  
b) Ожидается много запросов, но не очень большими данными.  
c) Строгий шаблон данных.  
  
10. _PromoDiscountsService_ -- REST.  
a) Кэш справочников.  
b) Ожидается много запросов, но не очень большими данными.  
c) Строгий шаблон данных.  
  
11. _CourierGeoService_ -- WebSocket (данные нужно получать мобилке, а серверу -- от курьера).  
  
## 3. Схема взаимодействия

_PNG/PlantUML с указанием протоколов (в diagrams/)._  
**Приложено в папке диаграмм**.  

## 4. API-контракты

_3–4 ключевых API (Swagger/OpenAPI или описание)._  
  
1. Orders  
REST.  

Базовый путь: `/api/v1/orders`

#### Создание заказа

- Метод: `POST`
- Путь: `/`
- Описание: клиент передаёт корзину и адрес. Сервис создаёт заказ в статусе `PENDING` и запускает сагу.
- Тело запроса (application/json):

```json
{
  "userId": "uuid",
  "restaurantId": "uuid",
  "items": [
    {
      "menuItemId": "uuid",
      "quantity": 2,
      "price": 250.50
    }
  ],
  "deliveryAddress": {
    "street": "Тверская ул., д. 1",
    "apartment": "45",
    "comment": "домофон 123"
  },
  "promoCode": "string (опционально)"
}  
```  
Response status code: 201 (Created):  
```JSON
{
  "orderId": "uuid",
  "status": "PENDING",
  "estimatedTotal": 550.00,
  "createdAt": "2026-07-20T10:00:00Z"
}
```  
Получение статуса заказа
Метод: GET

Путь: /{orderId}

Описание: используется клиентом для отображения текущего состояния.

Ответ (200 OK):

```json
{
  "orderId": "uuid",
  "status": "PREPARING",
  "courierId": "uuid (опционально)",
  "estimatedDeliveryTime": "2026-07-20T10:45:00Z",
  "currentLocation": {
    "lat": 55.7558,
    "lng": 37.6173
  }
}
```  
Отмена заказа
Метод: POST

Путь: /{orderId}/cancel

Описание: клиент отменяет заказ до передачи курьеру. Сервис меняет статус и публикует событие для возврата средств.

Ответ (200 OK):

```json
{
  "orderId": "uuid",
  "status": "CANCELLED",
  "refundStatus": "IN_PROGRESS"
}  
```  
2. PartnersService 
Базовый путь: /api/v1/partners/restaurants

Подача заявки на регистрацию
Метод: POST

Путь: /

Описание: представитель ресторана заполняет анкету. Заявка сохраняется и отправляется модератору асинхронно.

Тело запроса:
```
json
{
  "legalName": "ООО Ромашка",
  "brandName": "Суши-бар Сакура",
  "inn": "1234567890",
  "contactEmail": "sakura@mail.ru",
  "contactPhone": "+79991234567",
  "menu": [
    {
      "name": "Филадельфия",
      "description": "Лосось, сливочный сыр",
      "price": 400.00,
      "category": "РОЛЛЫ"
    }
  ]
}
```
Ответ (202 Accepted):
```
json
{
  "partnerApplicationId": "uuid",
  "status": "MODERATION_PENDING"
}
```  
3. ChatsService  
Протокол: WebSocket (wss://)

Путь подключения: /ws/chat/{orderId}?userId={userId}&role={client|courier|support}

Описание: устанавливается постоянное двустороннее соединение для обмена сообщениями между участниками заказа (клиент, курьер, поддержка). Все сообщения сохраняются в БД чатов и доставляются всем подключённым участникам диалога.

Формат входящего сообщения
```
json
{
  "type": "message",
  "payload": {
    "text": "Здравствуйте, где мой заказ?",
    "timestamp": "2026-07-20T10:15:00Z"
  }
}
```
Формат исходящего сообщения (от сервера клиенту)
```
json
{
  "type": "message",
  "payload": {
    "messageId": "uuid",
    "senderId": "uuid",
    "senderRole": "client",
    "text": "Здравствуйте, где мой заказ?",
    "timestamp": "2026-07-20T10:15:00Z"
  }
}
```
Дополнительные типы событий, которые сервер может отправлять:

"type": "connected" — подтверждение успешного подключения.

"type": "history" — массив последних сообщений при первом подключении.

"type": "error" — ошибка (например, нет прав на этот чат).
  
4. CourierGeoService  
Протокол: WebSocket (wss://)

Путь подключения (для курьера): /ws/courier/position?courierId={uuid}

Путь подключения (для клиента): /ws/order/tracking?orderId={uuid} — клиент подписывается на обновления геопозиции конкретного заказа.

Описание: курьер периодически отправляет свои GPS-координаты, сервер транслирует их всем клиентам, ожидающим этот заказ.

Формат входящего сообщения (от курьера)
```
json
{
  "type": "location_update",
  "payload": {
    "lat": 55.7558,
    "lng": 37.6173,
    "accuracy": 10,
    "timestamp": "2026-07-20T10:20:00Z"
  }
}
```
Формат исходящего сообщения (клиенту)

```
json
{
  "type": "location_update",
  "payload": {
    "courierId": "uuid",
    "lat": 55.7558,
    "lng": 37.6173,
    "timestamp": "2026-07-20T10:20:00Z"
  }
}
```
Сервер также может отправлять служебные сообщения:

"type": "assigned" — уведомление о том, что курьер назначен на заказ.

"type": "arrived" — курьер прибыл к ресторану или к клиенту.

"type": "error" — ошибка авторизации или некорректные данные.

## 5. Асинхронность  
  
1. _RestaurantsHandbooks_:  
a) Для получения данных мобилкой -- sync.  
b) Для записи -- sync.  
c) Актуализация справочников и кэша async worker.  
  
2. _OrdersService_:  
a) Запись в свою базу по обращению из внешнего эндпоинта - sync.  
b) Оповещение других микросервисов об обновлении данных -- async Kafka.  
c) Актуализация справочников и кэша -- async worker.  
  
3. _CouriersService_:  
a) Запись в свою базу по обращению из внешнего эндпоинта - sync.  
b) Оповещение других микросервисов об обновлении данных -- async Kafka.  
c) Актуализация справочников и кэша -- async worker. 
  
4. _ChatsService_:  
a) Запись в свою базу по обращению из внешнего эндпоинта - sync.  
b) Чтение данных из Kafka от других микросервисов - async.  
  
5. _NotificationService_:  
a) Чтение данных из Kafka от других микросервисов - async.  
b) Отправка интеграциям (пуши и т.д.) -- в зависимости от протокола интеграции. Скорее всего, sync REST?    
  
6. _PaymentService_:  
a) Со своей б/д -- sync.  
b) Актуализация справочников и кэша -- async worker.  
c) Внешние интеграции -- в зависимости от протокола интеграции. Скорее всего, sync REST? Тем более тут данные ПЛАТЕЖНЫЕ.  
  
7. _MotivationService_:  
a) Со своей б/д -- sync.  
b) Актуализация справочников и кэша -- async worker.  
c) Чтение данных из Kafka от других микросервисов - async.  
d) Расчет мотивационных надбавок -- async worker раз в N.  
  
8. _CalculatorService_:  
a) Со своей б/д -- sync.  
b) Актуализация справочников и кэша -- async worker. 
  
9. _PromoDiscountsService_:  
a) Со своей б/д -- sync.  
b) Актуализация справочников и кэша -- async worker.  
  
10. _CourierGeoService_:  
a) Двустороннее соединение WebSocket для получения и отдачи данных о курьере (от мобилки-1 (у курьера) для мобилки-2 (у заказчика)).  
b) Чтение данных из Kafka от других микросервисов - async.  
  
## 6. Паттерны  

6.1. Saga (хореография)
Заказ проходит через несколько независимых сервисов (Orders, Payment, Couriers, Notification). Чтобы сохранить целостность данных без распределённых транзакций, используется Saga в стиле хореографии.

Процесс создания заказа:

OrdersService получает запрос, сохраняет заказ со статусом PENDING и публикует событие OrderCreated в Kafka.

PaymentService читает событие, пытается списать средства через внешнюю платёжную систему. При успехе публикует PaymentSucceeded, при ошибке – PaymentFailed.

CouriersService читает PaymentSucceeded и ищет ближайшего свободного курьера. Если находит – назначает его и публикует CourierAssigned. Если нет – публикует CourierUnavailable.

OrdersService подписан на все эти события. Получив CourierAssigned, переводит заказ в ACTIVE. Получив PaymentFailed или CourierUnavailable, запускает компенсацию.

Компенсирующие действия:

OrdersService публикует событие OrderCompensationRequired.

PaymentService, получив его, инициирует возврат денег.

CouriersService освобождает курьера (если был назначен).

Такая схема обеспечивает слабую связанность: каждый сервис знает только о шине данных, а не о других сервисах напрямую.

6.2. Transactional Outbox
При асинхронном обмене через Kafka есть риск потерять событие, если сервис упадёт после записи в БД, но до отправки сообщения.

Решение – паттерн Transactional Outbox:

В каждом сервисе-генераторе событий (Orders, Payment, Couriers) есть таблица OutboxMessages в локальной реляционной БД.

Сохранение доменных данных и записи в outbox выполняется в одной ACID-транзакции. Например, в OrdersService при создании заказа одновременно вставляются строки в Orders и OutboxMessages со статусом PENDING.

Отдельный фоновый воркер периодически сканирует таблицу, выбирает неотправленные сообщения и отправляет их в Kafka. После подтверждения от брокера статус меняется на SENT или запись удаляется.

Это гарантирует, что каждое изменение состояния будет опубликовано, даже при сбоях инфраструктуры.

6.3. CQRS для справочников
Нагрузка на чтение и запись данных о ресторанах и меню существенно различается, поэтому применяется разделение ответственности (CQRS).

PartnersService отвечает за запись (Command): создание, обновление цен, модерацию. Это сложные операции с проверками, нагрузка относительно невысокая.

RestaurantsHandbooksService отвечает за чтение (Query): быстрая выдача каталога для мобильных клиентов. На этот сервис приходится основная read-нагрузка.

Синхронизация между ними происходит асинхронно через Kafka: как только PartnersService подтверждает изменение (например, одобрение нового меню), он публикует событие MenuUpdated. RestaurantsHandbooksService читает это событие и обновляет свою read-модель (инвалидирует кэш в Redis или обновляет денормализованные таблицы). Благодаря этому чтение и запись масштабируются независимо, а фронтенд получает ответы с минимальной задержкой.

Diagram code:  
@startuml FoodDelivery_Architecture
!theme plain
skinparam componentStyle rectangle
skinparam linetype ortho
skinparam nodesep 40
skinparam ranksep 50

title Схема взаимодействия сервисов — Доставка еды

actor "Клиент\n(Mobile App)" as Client
actor "Курьер\n(Mobile App)" as Courier
actor "Партнёр\n(Portal)" as Partner
actor "Модератор" as Moderator

rectangle "API Gateway" as Gateway #LightBlue

cloud "Внешние системы" {
  [Платёжные\nпровайдеры] as PayProv
  [APNs / FCM] as Push
  [Email SMTP] as Email
}

rectangle "Event Bus" {
  queue "Kafka\n(order-events,\npayment-events,\ncourier-events,\nmenu-events...)" as Kafka
}

' Группы сервисов
package "Каталог и партнёры" {
  [RestaurantsHandbooksService\n(REST)] as Catalog
  [PartnersService\n(REST)] as Partners
}

package "Основной поток заказа" {
  [OrdersService\n(REST + Kafka)] as Orders
  [PaymentService\n(REST + Kafka)] as Payments
  [CouriersService\n(REST + Kafka)] as Couriers
  [CalculatorService\n(REST)] as Calc
  [PromoDiscountsService\n(REST)] as Promo
}

package "Realtime" {
  [ChatsService\n(WebSocket)] as Chats
  [CourierGeoService\n(WebSocket)] as Geo
}

package "Вспомогательные" {
  [NotificationsService\n(Kafka → Push/Email)] as Notif
  [MotivationService\n(REST + Kafka)] as Motivation
}

' Клиенты → Gateway
Client --> Gateway : REST
Courier --> Gateway : REST / WebSocket
Partner --> Gateway : REST
Moderator --> Gateway : REST

' Gateway → сервисы (REST)
Gateway --> Catalog : REST
Gateway --> Orders : REST
Gateway --> Couriers : REST
Gateway --> Partners : REST
Gateway --> Payments : REST
Gateway --> Calc : REST
Gateway --> Promo : REST
Gateway --> Motivation : REST
Gateway --> Chats : WebSocket
Gateway --> Geo : WebSocket

' Kafka взаимодействия (основные)
Orders --> Kafka : publish OrderCreated / OrderUpdated
Payments --> Kafka : publish PaymentSucceeded / Failed
Couriers --> Kafka : publish CourierAssigned / Unavailable
Partners --> Kafka : publish MenuUpdated / PartnerApproved
Kafka --> Notif : consume → push/email
Kafka --> Orders : consume (статусы)
Kafka --> Couriers : consume
Kafka --> Catalog : consume (обновление read-модели)
Kafka --> Motivation : consume
Kafka --> Geo : consume
Kafka --> Chats : consume

' Внешние
Payments --> PayProv : REST (sync)
Notif --> Push : REST/API
Notif --> Email : SMTP/API

' CQRS-связь
Partners -[dashed]-> Catalog : через Kafka\n(MenuUpdated)

note right of Kafka
  Основные топики:
  • order-events
  • payment-events
  • courier-events
  • menu-events
  • notification-events
end note

@enduml
