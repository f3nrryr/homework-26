# ДЗ 3. Проектирование хранения данных — решение

> Условие: [../Задание.md](../Задание.md)  
> Шаблоны: [сравнение БД](../Таблица%20сравнения%20БД%20с%20критериями%20выбора.md) · [кэширование](../Чек-лист%20стратегий%20кэширования.md) · [шардирование](../Шаблон%20схемы%20шардирования.md)  
> Схемы — в [diagrams/](diagrams/).  
> Выполняется для системы, выбранной в ДЗ 2 (Доставка еды).

## 1. Модель данных

### RestaurantsHandbooksService (read-модель)
- **restaurants**: id, name, description, address, rating, is_active, updated_at
- **menu_items**: id, restaurant_id, name, description, price, category, photo_url, is_available, version
- **categories**: id, name, sort_order

Основной доступ — по restaurant_id и спискам позиций. Данные денормализованы для быстрого чтения.

### PartnersService (Command + runtime)
- **partner_applications**: id, legal_name, brand_name, inn, contact, status, created_at
- **menu_versions**: id, restaurant_id, items (jsonb), status (draft/moderation/approved), moderated_by
- **restaurant_orders**: order_id, restaurant_id, status (pending/accepted/rejected), accepted_at, reject_reason

### OrdersService
- **orders**: id, user_id, restaurant_id, status, pricing_snapshot (jsonb), delivery_address (jsonb), created_at, updated_at
- **order_items**: id, order_id, menu_item_id, name, quantity, price_at_order
- **order_status_history**: id, order_id, status, changed_at, reason

### PricingService
- **pricing_rules**: id, type (delivery/service_fee), conditions (jsonb), value, active_from, active_to
- **promo_codes**: code, discount_type, value, max_uses, used_count, valid_until, restaurant_id (nullable)

### PaymentsService
- **payments**: id, order_id, user_id, amount, currency, status (authorized/captured/refunded/failed), provider_payment_id, created_at
- **payment_events**: id, payment_id, event_type, payload, created_at
- **courier_payouts**: id, courier_id, amount, period, status

### CouriersService
- **couriers**: id, user_id, status (online/offline/busy), current_order_id, rating, vehicle_type
- **courier_assignments**: id, order_id, courier_id, assigned_at, completed_at

### CourierGeoService
- **courier_locations** (горячие): courier_id, lat, lng, accuracy, updated_at (хранится в Redis GEO)
- **location_history** (опционально): courier_id, lat, lng, recorded_at (time-series)

### ChatsService
- **conversations**: id, order_id, participants (array), created_at
- **messages**: id, conversation_id, sender_id, sender_role, text, created_at, is_read

### NotificationsService
- **notification_templates**: id, type, channel, body_template
- **notification_log**: id, user_id, order_id, channel, status, sent_at

## 2. Выбор БД

| Сервис | Что хранит | Форма данных | Ключевые запросы | Выбранная БД | Почему именно она |
|--------|------------|--------------|------------------|--------------|-------------------|
| OrdersService | заказы, позиции, история статусов | реляционная, жёсткие связи | по order_id, по user_id, смена статуса | **PostgreSQL** | Нужны ACID-транзакции при создании заказа и смене статусов, JOIN-ы, целостность |
| PaymentsService | платежи, hold/capture, выплаты | реляционная, деньги | по order_id, по payment_id, аудит | **PostgreSQL** | Критичная консистентность денежных операций, строгий аудит, ACID |
| PartnersService | заявки, меню, accept/reject | реляционная | по restaurant_id, статусы заявок | **PostgreSQL** | Транзакции при модерации и принятии заказа, связи |
| CouriersService | курьеры, назначения | реляционная | по courier_id, свободные курьеры | **PostgreSQL** | CRUD + статусы, относительно небольшой объём, нужны связи |
| RestaurantsHandbooksService | каталог ресторанов и меню | реляционная + денормализация | по restaurant_id, списки позиций, поиск | **PostgreSQL + Redis** | PostgreSQL — источник истины; Redis — горячий кэш read-модели (очень высокая read-нагрузка) |
| PricingService | правила расчёта, промокоды | key-value / небольшие таблицы | расчёт по параметрам заказа | **Redis** (+ PostgreSQL для долговременных правил) | Очень частые чтения, маленький объём данных, критична низкая latency |
| CourierGeoService | текущие координаты курьеров | key-value + geo | «ближайшие курьеры», обновление позиции | **Redis (GEO)** | Тысячи записей в секунду, нужны быстрые geo-запросы (GEORADIUS), короткая жизнь данных |
| ChatsService | сообщения чатов | документы | по conversation_id / order_id, последние N сообщений | **MongoDB** | Высокая скорость записи, сообщения как самодостаточные документы, редко нужны JOIN-ы |
| NotificationsService | шаблоны + лог отправок | простые записи | по user_id / order_id | **PostgreSQL** или **Redis** | Небольшой объём, не критично; PostgreSQL — если нужен долгий аудит |

## 3. Шардирование

Шардирование требуется для сервисов с высоким объёмом записи и ростом данных.

### 3.1. OrdersService

| Параметр | Решение |
|----------|---------|
| Что шардируем | orders + order_items + order_status_history |
| Ключ шардирования | `user_id` (hash) |
| Стратегия | Hash-шардирование (consistent hashing) |
| Почему этот ключ | Большинство запросов — «история заказов пользователя» и «мои активные заказы». Они попадают в один шард. Запросы по `order_id` решаются через небольшой lookup-сервис или глобальный вторичный индекс. |
| Стартовое количество шардов | 4–8 |
| Ребалансировка | Consistent hashing + virtual nodes — при добавлении шарда переезжает только часть ключей |
| Главный риск | Hotspot на очень активных пользователях. Лечение: соль к ключу (`user_id + salt`) или переход на `hash(order_id)` + отдельный индекс для user-history |
| Cross-shard | Сведены к минимуму: связанные данные заказа лежат в одном шарде -> локальные транзакции возможны |

### 3.2. CourierGeoService

| Параметр | Решение |
|----------|---------|
| Что шардируем | текущие координаты курьеров |
| Ключ шардирования | `courier_id` (hash) |
| Стратегия | Hash + Redis Cluster / шардированный Redis |
| Почему | Обновления идут по конкретному курьеру. Запросы «ближайшие» выполняются через Redis GEO (можно держать несколько geo-индексов или использовать proxy). |
| Риск | При очень большом количестве курьеров — переход на гео-шардирование (по geohash/региону) для лучшей локальности запросов «рядом со мной». |

## 4. Кэширование

| Что кэшируем | Где | Стратегия | TTL | Инвалидация |
|--------------|-----|-----------|-----|--------------|
| Каталог ресторанов и меню | Redis | Cache-Aside | 5–15 мин | По событию `MenuUpdated` из Kafka (PartnersService) |
| Результаты расчёта цены (по параметрам) | Redis | Cache-Aside | 1–3 мин | Короткий TTL + инвалидация при изменении правил |
| Текущие координаты курьеров | Redis GEO | Write-Through / прямая запись | 30–60 сек | Автоматически устаревают |
| Статус заказа (для частых опросов клиента) | Redis | Cache-Aside | 10–30 сек | По событиям смены статуса |
| Промокоды (активные) | Redis | Cache-Aside | 5 мин | При изменении/исчерпании |

**Что сознательно не кэшируем:**
- Платёжные данные и статусы hold/capture (критична актуальность)
- Принятие/отклонение заказа рестораном (должно быть максимально свежим)
- Сообщения чата (идут через WebSocket, кэш не нужен)

## 5. Очереди

Используется **Apache Kafka** (уже выбран в ДЗ 2).

**Где применяется:**
- Публикация доменных событий саги (`OrderCreated`, `RestaurantAccepted`, `PaymentAuthorized`, `CourierAssigned` и т.д.)
- Синхронизация CQRS (Partners -> RestaurantsHandbooks через `MenuUpdated`)
- Асинхронная отправка уведомлений (NotificationsService)
- Обновление геопозиции и производные события

**Почему именно Kafka:**
- Высокий throughput (нужен при пиковых нагрузках)
- Гарантия порядка внутри партиции (partition key = `orderId`)
- Retention — можно перечитывать события при восстановлении
- Хорошо подходит для event-driven архитектуры и саги
- Поддержка consumer groups и горизонтального масштабирования консьюмеров

Дополнительно: для очень короткоживущих real-time сигналов (например, мгновенная доставка location_update) можно использовать Redis Pub/Sub, но основным брокером остаётся Kafka.

## 6. CAP trade-offs

| Сервис / данные | Выбор | Обоснование |
|-----------------|-------|-------------|
| OrdersService + PaymentsService | **CP** | Деньги и статусы заказа не могут быть «примерно правильными». При сетевом разделении предпочитаем отказать в операции, чем отдать неконсистентные данные. |
| RestaurantsHandbooksService (каталог) | **AP** | Доступность каталога для клиентов важнее мгновенной согласованности цен. Eventual consistency через CQRS + Kafka допустима. При создании заказа цена дополнительно перепроверяется. |
| CourierGeoService | **AP** | Координаты могут быть на несколько секунд устаревшими. Главное — система продолжает отвечать и показывать курьера. |
| ChatsService | **AP** | Доступность чата важнее строгой консистентности порядка сообщений в редких edge-cases. |
| PricingService | ближе к **CP** при расчёте | Расчёт цены выполняется синхронно и должен быть точным. Кэш при этом может быть AP с коротким TTL. |
| NotificationsService | **AP** | Уведомление можно доставить с небольшой задержкой, главное — не потерять и не заблокировать основной поток. |

**Итоговая стратегия:**  
Критические пути (деньги, создание и смена статуса заказа) — **Consistency + Partition tolerance**.  
Всё, что связано с отображением и real-time (каталог, гео, чат, пуши) — **Availability + Partition tolerance** с управляемой eventual consistency.
