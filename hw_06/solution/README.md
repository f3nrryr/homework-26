# Курсовой проект. Мессенджер с групповыми чатами

> Аналог Telegram. Система **не** совпадает с ДЗ 2–5 (доставка еды).  
> Схемы — в [diagrams/](diagrams/). ADR — в этом документе.  

---

# Часть A. Требования и концептуальная архитектура

## 1. Требования

### 1.1. Функциональные требования (scope)

1. Регистрация и вход (телефон / email), **сессии на нескольких устройствах** (у каждого устройства свой cursor, см. §4.5).
2. Профиль, поиск пользователей по username, список контактов.
3. Личные диалоги 1-to-1.
4. Групповые чаты до **200 участников**: создание, приглашение, роли admin/member, выход.
5. Отправка и получение текстовых сообщений, edit/delete у себя, reply, last-seen.
6. Вложения: фото и файлы до 20 МБ (видеозвонки и стриминг — вне scope).
7. Онлайн-статус и «печатает».
8. Push на мобильные, если ни одно устройство пользователя не держит живой WebSocket.
9. Синхронизация истории при входе с нового устройства (окно **1 год** горячей истории) + догон с watermark устройства.
10. **Непрочитанные и receipts** (упрощённо): watermark `last_read` на пару (user, chat, device); в личке — delivered/read; в группе — без «кто прочитал» на каждое сообщение.

### 1.2. Нефункциональные требования

| Метрика | Значение |
|---------|----------|
| MAU | 10 000 000 |
| DAU | 2 000 000 |
| Сообщений отправлено на DAU в день | 30 |
| Доля сообщений с вложением | 10%, среднее вложение 200 КБ |
| Пик | ×3 вечером (20:00–23:00) |
| p99 доставки сообщения онлайн-получателю | < 500 мс в том же регионе |
| p95 открытия списка чатов | < 200 мс |
| Доступность контура отправки | 99,9% |
| RPO сообщений при отказе ноды/AZ | секунды (QUORUM) |
| RPO сообщений при отказе региона | минуты (снимок + replay) |
| RTO отправки (нода/AZ) | 10 мин |
| История | 1 год в горячем хранилище, старше — cold |
| Макс. размер группы в scope | 200 |

Формулы нагрузки:

```
сообщений_в_день = DAU × 30 = 2 000 000 × 30 = 60 000 000
avg_send_rps     = 60 000 000 / 86 400 ≈ 694
peak_send_rps    = 694 × 3 ≈ 2 080
```

Чтение (inbox + история при открытии чата) принимаем **×8** к отправке:

```
peak_read_rps ≈ 16 600
```

### 1.3. Риски и ограничения

- Медиа и egress — главная статья счёта; без CDN проект экономически не сходится.
- Группа на 200 человек × пик даёт fan-out. Без fan-out on write на всех членов один чат не обязан плодить 200 копий тела.
- Нельзя обещать E2EE «как в Telegram Secret Chats» без отдельного протокола — не берём.
- Команда условно небольшая: не плодим 20 сервисов.
- Один регион + DR warm standby. Multi-region active-active для сообщений — конфликт порядка и дорого.
- Unread в группе **не** считаем инкрементом на каждого члена при каждой отправке — это отдельный hotspot.

### 1.4. Backlog (не в scope)

- Каналы и супергруппы 200k+
- Голосовые/видеозвонки, кружки, stories
- Секретные чаты с E2EE
- Боты-платформа и Mini Apps
- Полнотекстовый глобальный поиск по всем чатам пользователя (в MVP — поиск по открытому чату)
- Мультирегион active-active
- Read-receipts «кто именно прочитал» в групповом чате (список галочек на 200 человек)

---

## 2. Концептуальная архитектура

### 2.1. Стиль

**Микросервисы + event-driven на шине доменных событий**, синхронный путь только там, где клиенту нужен ответ сейчас (отправка сообщения, список чатов, загрузка файла).

Почему не монолит: разные профили нагрузки (короткие сообщения vs медиа vs presence vs push) и разные требования к консистентности.  
Почему не 15 сервисов: Contacts не отделяем от Users, edit/delete не отделяем от Messages. Реестр WS-сессий живёт в Redis у Presence, отдельный SessionService не заводим.

### 2.2. Пользователи и внешние системы (C4 Context)

Акторы: пользователь (mobile / web / desktop), администратор группы.

Внешние: IdP / SMS-шлюз, APNs, FCM, object storage + CDN, антиспам (опционально).

### 2.3. Контейнеры (C4 Container)

| Контейнер | Ответственность |
|-----------|-----------------|
| API Gateway / BFF | TLS, JWT, rate limit, REST + **терминация WebSocket**. На коннекте регистрирует сессию в Redis |
| AuthService | регистрация, OTP, сессии, refresh, список устройств |
| UsersService | профиль, username, контакты |
| ChatsService | диалоги и группы: состав, роли, last_message pointer |
| MessagesService | запись/чтение сообщений. **Не** держит WS и **не** знает ноду получателя |
| MediaService | upload URL, метаданные файла |
| PresenceService | online / typing + **маршрутизация доставки**: читает MessageSent, смотрит реестр сессий, публикует в канал нужного Gateway |
| PushService | пуши, если в реестре нет живых сессий |
| Kafka | MessageSent, MemberAdded, ReceiptUpdated |
| Redis | presence, кэш, **реестр WS-сессий**, pub/sub на `gw:{node}` |
| PostgreSQL | users, chats, auth, devices |
| Cassandra | messages_by_chat, inbox_by_user, read_watermarks |
| S3 + CDN | файлы |

Схема: [diagrams/C4_Container.puml](diagrams/C4_Container.puml).

### 2.4. Как сообщение доходит до WebSocket (дыра закрыта)

`MessagesService` после записи в Cassandra **не вызывает Gateway**. Цепочка:

1. Клиент открывает WS на конкретном поде Gateway. Под пишет в Redis:
   `HSET ws:user:{user_id} {device_id} → {gateway_node, conn_id, ts}`  
   и подписан на канал `gw:{свой_node_id}`.
2. `MessagesService` пишет строку в `messages_by_chat` и публикует `MessageSent` в Kafka (`key = chat_id`).
3. `PresenceService` (консьюмер fan-out) берёт получателей: в личке — второй участник; в группе — состав из кэша чата.
4. Для каждого `user_id` читает реестр сессий в Redis.
5. Если есть живые сессии — `PUBLISH gw:{gateway_node}` с `conn_id` и телом события. Нужный под Gateway пушит в свой сокет.
6. Если сессий нет ни на одном устройстве — событие уходит в `PushService`.

Несколько устройств одного пользователя = несколько записей в hash. Событие уходит на все живые сокеты; у каждого устройства свой watermark (§4.5).

Это **ADR-4**: реестр сессий в Redis + pub/sub по ноде Gateway, не sticky-маршрутизация по cookie и не прямой gRPC Messages → Gateway (иначе узнавать ноду пришлось бы из Messages, и при рестарте пода реестр врал бы).

### 2.5. ADR

**ADR-1. Хранение сообщений в Cassandra, не в PostgreSQL**  
Контекст: 60 млн сообщений/сутки, чтение по `(chat_id, message_id)`.  
Решение: wide-column, PK `((chat_id), message_id)`.  
Отвергнуто: PostgreSQL на все сообщения.  
Цена: нет JOIN «сообщение × пользователь»; имя отправителя денормализуем.

**ADR-2. Fan-out on write для личных чатов, fan-out on read для групп**  
Контекст: 1-to-1 vs группа 200.  
Решение: в личке — строка в `inbox_by_user`; в группе — одна строка в партиции чата.  
Цена: unread группы считаем через watermark, не через 200 инкрементов.

**ADR-3. Не E2EE в MVP**  
TLS + at-rest. E2EE — backlog.

**ADR-4. Доставка онлайн через реестр сессий, не из Messages**  
См. §2.4.

---

# Часть B. Сайзинг, данные и взаимодействие

## 3. Сайзинг

### 3.1. RPS

| Поток | Событий / день | Avg RPS | Peak ×3 |
|-------|----------------|---------|---------|
| Отправка сообщения | 60 000 000 | 694 | **2 080** |
| Чтение истории / sync | 480 000 000 | 5 556 | **16 700** |
| Список чатов | 2 000 000 × 20 = 40 000 000 | 463 | **1 390** |
| Presence heartbeat | 2 000 000 × 200 ≈ 400 000 000 | 4 630 | **13 900** |
| Upload медиа | 6 000 000 | 69 | **210** |
| Push | ~0,3 на сообщение офлайн ≈ 20 000 000 | 231 | **700** |

Узкие места: **чтение истории** и **presence**, не «создать чат».

### 3.2. Хранилище

Сообщение в Cassandra ≈ 2 КБ (текст + метаданные, без блоба):

```
60e6 × 2 КБ × 365 ≈ 44 ТБ / год сырых
× RF=3 ≈ 132 ТБ данных на дисках кластера
```

Горячий год держим. Старше 12 мес — cold / TTL.

Медиа:

```
6e6 вложений/день × 200 КБ ≈ 1,2 ТБ / день
за год ≈ 438 ТБ сырых
реплика object storage ×2 ≈ 876 ТБ
```

### 3.3. Bandwidth

Текст пика: 2 080 × 2 КБ ≈ 4 МБ/с — шум.

Медиа без CDN:

```
скачиваний в день ≈ загрузки × 4
6e6 × 4 × 200 КБ ≈ 4,8 ТБ / день
×30 ≈ 144 ТБ / мес egress
```

С CDN offload 90%:

```
origin = 0,10 × 144 = 14,4 ТБ / мес ≈ 14 ТБ
edge   = 0,90 × 144 = 130 ТБ / мес
```

### 3.4. Ресурсы и стоимость

**Cassandra — пересчёт ёмкости.**  
132 ТБ уже *с* репликацией. Compaction требует, чтобы диск не был забит: целевое заполнение **50–55%**.

```
нужный сырой диск = 132 ТБ / 0,55 ≈ 240 ТБ
узлов при диске 8 ТБ: 240 / 8 = 30
заполнение: 132 / 240 = 55%
```

12 узлов × 8 ТБ = 96 ТБ < 132 ТБ даже без запаса на compaction — в предыдущей версии смета была занижена. Берём **30 узлов × 8 ТБ**.  
Альтернатива: 24 узла × 16 ТБ (тот же запас, меньше машин, дороже диск).

Строка Cassandra в смете: было $12 000 на 12 узлов → линейно **$30 000** на 30 узлов.

**CDN, без подгона:**

```
14 000 ГБ × $0,08  = $1 120   origin
130 000 ГБ × $0,03 = $3 900   edge
итого сеть с CDN   = $5 020
144 000 ГБ × $0,08 = $11 520  сеть без CDN
```

**Reserved −30%** только на compute + управляемые БД (стабильная база 24/7). На egress скидки reserved нет.

| Статья | On-demand $/мес | После reserved |
|--------|-----------------|----------------|
| Compute ~150 vCPU | 4 500 | 3 150 |
| Cassandra 30 узлов | 30 000 | 21 000 |
| PostgreSQL HA | 1 500 | 1 050 |
| Redis cluster | 1 200 | 840 |
| Kafka + RabbitMQ | 1 800 | 1 260 |
| S3 hot 200 ТБ + cold 700 ТБ | 7 500 | 7 500 |
| Egress с CDN | 5 020 | 5 020 |
| **Сумма on-demand + CDN** | **51 520** | |
| **Сумма с reserved на железо** | | **39 820** |
| Egress без CDN вместо 5 020 | +6 500 | итог без CDN ≈ **46 300** с reserved |

```
Cost / DAU (с CDN и reserved) ≈ 39 800 / 2 000 000 ≈ $0,020 / мес
```

Раньше $28 000 получались из заниженной Cassandra и округления CDN. Честная цифра — **~$40k** с оптимизациями и **~$52k** on-demand.

---

## 4. Хранение данных

### 4.1. Выбор БД

| Сервис | БД | Почему |
|--------|----|--------|
| Auth, Users, Chats, devices | PostgreSQL | связи, роли, уникальный username, транзакция «создать группу + админ» |
| Messages | Cassandra | append по чату, предсказуемый ключ, TTL |
| Inbox 1-to-1 | Cassandra | PK по `user_id` |
| Watermark непрочитанных | Cassandra | PK `(user_id, chat_id)` |
| Presence + реестр WS | Redis | TTL, pub/sub |
| Кэш списка чатов | Redis | Cache-Aside |
| Идемпотентность send | Redis | `client_msg_id` 24 ч |
| Медиа-метаданные | PostgreSQL | file_id → s3 key |
| Блобы | S3 + CDN | |

### 4.2. Шардирование

**Шаг 0.** PG users/chats **не шардируем**: 10 млн профилей × 2 КБ ≈ 20 ГБ.

**Messages — сразу.** 2k write RPS + 44 ТБ/год сырых на одну ноду не кладём.

| Что | Ключ | Стратегия |
|-----|------|-----------|
| `messages_by_chat` | `chat_id` | token-aware Cassandra |
| `inbox_by_user` | `user_id` (+ bucket по месяцу) | hash |
| реестр WS / presence | `user_id` | Redis Cluster |

Не шардируем ленту по `user_id`: группа иначе scatter-gather.

`message_id` сам несёт `chat_id` — lookup не нужен (§4.4).

### 4.3. Кэш

| Уровень | Что | TTL | Инвалидация |
|---------|-----|-----|-------------|
| CDN | фото/файлы | часы–сутки | content-hash в URL |
| Redis | список чатов | 30–60 с | MessageSent / MemberChanged |
| Redis | состав группы | 1–5 мин | MemberChanged |
| Redis | профиль | 5 мин | UpdateProfile |
| Redis | presence + ws-sessions | 30–60 с | heartbeat / disconnect |
| Не кэшируем | тело сообщения как истину | — | Cassandra; last page чата — 10 с |

### 4.4. Схемы данных (CQL / DDL)

Формат `message_id` (128 бит, ULID-подобный):

```
[ 48 бит chat_id_prefix ][ 48 бит unix_ms ][ 32 бит worker+seq ]
```

По id достаём префикс чата и попадаем в ту же партицию, что и `chat_id`. Клиент в API всё равно передаёт `chatId`.

```cql
CREATE TABLE messages_by_chat (
    chat_id     uuid,
    message_id  timeuuid,
    sender_id   uuid,
    sender_name text,          -- денормализация
    type        text,          -- text | file
    body        text,
    file_id     uuid,
    reply_to    timeuuid,
    created_at  timestamp,
    PRIMARY KEY ((chat_id), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC)
  AND default_time_to_live = 31536000;   -- 1 год горячий

CREATE TABLE inbox_by_user (
    user_id     uuid,
    month       int,           -- YYYYMM, чтобы партиция не росла бесконечно
    message_id  timeuuid,
    chat_id     uuid,
    preview     text,
    PRIMARY KEY ((user_id, month), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

CREATE TABLE read_watermarks (
    user_id          uuid,
    chat_id          uuid,
    device_id        uuid,
    last_read_id     timeuuid,
    last_read_at     timestamp,
    PRIMARY KEY ((user_id), chat_id, device_id)
);
```

PostgreSQL (коротко):

```sql
CREATE TABLE devices (
    device_id    uuid PRIMARY KEY,
    user_id      uuid NOT NULL,
    platform     text,
    push_token   text,
    last_seen_at timestamptz
);

CREATE TABLE sessions (
    session_id   uuid PRIMARY KEY,
    user_id      uuid NOT NULL,
    device_id    uuid NOT NULL,
    refresh_hash text NOT NULL
);
```

### 4.5. Unread, receipts, мультидевайс

**Личка.** После записи в `messages_by_chat` пишем указатель в `inbox_by_user` обоим. `delivered` — Presence увидел живую сессию или Push подтвердил. `read` — клиент шлёт `POST /chats/{id}/read` с `last_read_id`; обновляем watermark устройства и шлём `ReceiptUpdated` второму участнику.

**Группа.** Тело сообщения одно. Не делаем 200 инкрементов unread и не храним 200 receipts на каждое «ок».  
`unread ≈ max_message_id(chat) − last_read_id(user, chat, device)` считаем при `GET /chats` (max_id чата лежит в ChatsService / Redis).  
«Кто прочитал в группе» — backlog.

**Мультидевайс.** У каждого `device_id` свой watermark. Телефон прочитал — бейдж на телефоне гаснет; десктоп, который не открывал чат, остаётся с своим счётчиком.  
Новое устройство: сессия в `devices`, история за год из `messages_by_chat` по списку чатов пользователя, watermark стартует с «сейчас» (не тащим непрочитанные за 11 месяцев на новый телефон — иначе бейдж в тысячи). Догон живого потока — с `last_read_id` после reconnect: `GET ...?after={watermark}`.

---

## 5. Взаимодействие

### 5.1. Протоколы

| Грань | Протокол | Почему |
|-------|----------|--------|
| Клиент ↔ Gateway, команды | REST / HTTPS | отправка, история, read, профили |
| Клиент ↔ Gateway, живой канал | WebSocket | новые сообщения, typing, presence, receipts |
| Gateway ↔ сервисы | gRPC | низкая задержка |
| Доменные события | Kafka, key=`chat_id` | порядок в чате |
| Fan-out на конкретный под Gateway | Redis pub/sub `gw:{node}` | реестр сессий |
| Пуши с retry | RabbitMQ | delayed retry |
| Файл | signed PUT S3 / GET CDN | |

### 5.2. Схема доставки

См. [diagrams/C4_Container.puml](diagrams/C4_Container.puml) и [diagrams/Sequence_SendMessage.puml](diagrams/Sequence_SendMessage.puml).

```
Client --WSS--> Gateway-N          REGISTER Redis ws:user
Client --REST--> Gateway --> Messages --> Cassandra
Messages --Kafka--> Presence
Presence --GET Redis--> sessions[user]
Presence --PUB gw:N--> Gateway-N --WSS--> Client
нет сессий --> Push --> APNs/FCM
```

### 5.3. API-контракты

**Отправить сообщение**  
`POST /api/v1/chats/{chatId}/messages`  
Headers: `Authorization`, `Idempotency-Key` (= `client_msg_id`)

```json
{ "type": "text", "text": "Привет", "replyTo": null }
```

`201`:

```json
{
  "messageId": "a1b2c3d4-01J8K...",
  "chatId": "a1b2c3d4-....",
  "senderId": "u9",
  "createdAt": "2026-09-03T10:00:00Z",
  "status": "accepted"
}
```

Ошибки: 401, 403 (не член), 409 (повтор ключа — вернуть старое id), 413, 429.

**История**  
`GET /api/v1/chats/{chatId}/messages?before={messageId}&after={watermark}&limit=50`

**Прочитано**  
`POST /api/v1/chats/{chatId}/read`  
`{ "deviceId": "...", "lastReadId": "..." }`

**Создать группу**  
`POST /api/v1/chats` → `{ "type": "group", "title": "...", "memberIds": [] }`

**WS входящее:** `message.created`, `receipt.updated`, `presence`, `typing`.

**Загрузка файла:** `POST /api/v1/media` → signed URL.

---

# Часть C. Надёжность, безопасность, защита

## 6. Надёжность

### 6.1. RTO / RPO — два контура отказа

| Сервис | Отказ ноды / одной AZ | Отказ региона |
|--------|------------------------|---------------|
| | **RTO** / **RPO** | **RTO** / **RPO** |
| Messages | 10 мин / **секунды** (RF=3 QUORUM) | 30–60 мин / **минуты** (снимок + replay WAL/commitlog, лаг async DC) |
| Auth / Sessions | 10 мин / секунды | 30 мин / минуты |
| Chats / Users | 15 мин / минуты | 45 мин / минуты |
| Media origin | 30 мин / минуты | 30 мин / минуты (CDN и реплика бакета ещё живы) |
| Presence / реестр WS | 5 мин / минуты (перерегистрация с heartbeat) | 15 мин / минуты |
| Push | 30 мин / минуты | 30 мин / минуты |

Секундный RPO из §1 — это **не** обещание на падение всего региона. На регионе сознательно деградируем: Warm Standby не даёт RPO≈0 без active-active и конфликта seq.

### 6.2. Репликация

| Система | Как | RPO на практике |
|---------|-----|-----------------|
| Cassandra | RF=3, QUORUM read/write | ≈ 0 при живом кворуме в регионе |
| PostgreSQL | primary + sync replica в другой AZ | ≈ 0 на коммите внутри региона |
| Redis | cluster + replica | минуты; сессии перерегистрируются |
| Kafka | RF=3, min.isr=2 | ≈ 0 для подтверждённых |
| S3 | версионирование + реплика бакета в DR | минуты |

Реплика ≠ бэкап. Снапшот PG + snapshot Cassandra. Restore-тест раз в квартал.

### 6.3. DR региона

Pilot Light / Warm Standby: async-реплика PG, реплика бакета, холодные ноды Cassandra под restore.

Падение региона: promote PG, **восстановить Cassandra из снимка + replay** (RPO минуты — это норма для выбранной схемы, не баг таблицы §6.1), переключить DNS. Реестр WS строится заново: клиенты реконнектятся, Gateway снова сделает `HSET`. Медиа — из реплики бакета / CDN.

Не active-active: два порядка сообщений в одном чате.

## 7. Безопасность

- OIDC / JWT, короткий access, refresh **на устройство**.
- `userId` только из токена.
- Член чата видит историю; кикнуть может admin.
- Rate limit на `user_id` и `chat_id`.
- mTLS внутри, секреты в vault.
- TLS; диски и бакеты at rest.
- Не логируем текст сообщений — только ids и длины.

## 8. Observability

| Сервис / ресурс | Метод | Метрика | Зачем |
|-----------------|-------|---------|-------|
| Gateway | RED | RPS send / history | сверка с сайзингом |
| Gateway | RED | 5xx vs 4xx vs 429 | отказ vs клиент vs лимитер |
| Messages send | RED | p99 < 500 мс | SLO accept |
| Messages read | RED | p95 history | открытие чата |
| Presence fan-out | RED | p99 Redis lookup + publish | дыра доставки |
| Presence | RED | heartbeat rate | «все офлайн» |
| Cassandra | USE | pending compact, disk % | compaction / место |
| PG | USE | pool, replica lag | |
| Kafka | USE | consumer lag Presence/Push | доставка отстаёт |
| Redis | USE | evicted, размер `ws:user:*` | вымыло сессии |
| S3/CDN | USE | origin vs edge hit | egress $ |
| Бизнес | golden | accepted / мин | |
| Бизнес | golden | p99 до WS получателя | |
| Бизнес | golden | доля send без живой сессии → push | |

Алерты:

| Алерт | Порог | Sev | Реакция |
|-------|-------|-----|---------|
| Провал send | accepted/мин < 50% 3 мин | crit | Cassandra / деплой |
| p99 send > 800 мс 5 мин | SLO | crit | диск / QUORUM |
| 5xx Gateway > 2% 3 мин | отказ | crit | откат |
| lag Kafka Presence > 2 мин | онлайн не доходит | crit | consumer |
| Redis session miss spike | pub/sub мимо | warn | реестр / TTL |
| Cassandra disk > 70% | compaction не влезет | crit | ноды / TTL |
| CDN origin всплеск | деньги | warn | cache key |

## 9. Тестирование

**Нагрузка:** пик 2 080 send + 16 700 history; горячая группа 200 × 20 msg/s; soak 2 ч.

Успех: p99 send < 500 мс, 5xx < 0,5%, p99 fan-out (Kafka→WS) < 500 мс, диск Cassandra < 55% после soak.

**Chaos**

```
Эксперимент №1: Kill одной ноды Cassandra
Steady state:    accepted/мин в норме, p99 send < 500 мс
Гипотеза:        RF=3 QUORUM переживёт потерю 1 из 30 без роста 5xx
Blast radius:    staging, 1 нода
Метод:           stop на 10 мин
Наблюдаем:       p99 send, 5xx, unavailable
Rollback:        5xx > 2% 2 мин
Ожидаемый вывод: либо кворум жив, либо CL выставлен неверно
```

```
Эксперимент №2: Отказ APNs/FCM
Steady state:    онлайн-доставка по WS не деградирует
Гипотеза:        circuit + RabbitMQ; WS-путь не зависит от пушей
Blast radius:    staging, только egress push
Метод:           blackhole 15 мин
Наблюдаем:       p99 send, глубина RabbitMQ
Rollback:        5xx send > 1%
Ожидаемый вывод: офлайн отложен, онлайн жив
```

```
Эксперимент №3: Рестарт пода Gateway с живыми WS
Steady state:    p99 доставки до получателя < 500 мс
Гипотеза:        клиент реконнектится на другой под, реестр Redis
                 обновится, Presence не шлёт в мёртвый gw_node
Blast radius:    staging, 1 под Gateway из N
Метод:           kill пода на 5 мин
Наблюдаем:       session miss, дубли сообщений, p99 fan-out
Rollback:        потерянные online-доставки > 5%
Ожидаемый вывод: либо реестр консистентен с живыми сокетами,
                 либо залипли stale gateway_node
```

---

## Согласованность частей

- Messages не знает ноду Gateway — знает Presence + Redis.
- 132 ТБ с RF и 55% fill дают **30 узлов**, не 12; смета Cassandra $30k.
- RPO «секунды» — нода/AZ; RPO «минуты» — регион. Это одна политика на двух уровнях, не противоречие.
- CQL фиксирует ключи, которые уже были в шардировании.
- Unread в группе — watermark, не 200 запись на сообщение.
- Мультидевайс — `device_id` в сессии и в `read_watermarks`.
- CDN: $5 020, не $4 300. Итог с reserved ≈ $40k, без подгона 28k.
