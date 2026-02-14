# Draft: Sync Contract (facts-only)

Ограничения: этот документ составлен **строго** по материалам из `docs/analysis/network_facts.md` и `docs/analysis/roast_log_structures.md`.  
Если данных недостаточно, это помечено как **DATA GAP**. Любые предложения помечены как **RECOMMENDATION**.

## Facts

- **Транспорт/протоколы, найденные в коде**
  - **Artisan+/sync**: HTTPS JSON (`src/plus/connection.py`, `src/plus/sync.py`, `src/plus/queue.py`, `src/plus/stock.py`).
  - **Прочие bidirectional каналы**: WebSocket `ws://` (клиент `src/artisanlib/wsport.py`, сервер aiohttp `src/artisanlib/weblcds.py`).
  - **MQTT/AMQP/gRPC/SSE**: помечены как **NOT FOUND** в анализе.

- **Auth flow (как есть в коде)**
  - **Логин**: отправка на `config.auth_url` с JSON `{'email': aw.plus_account, 'password': config.passwd}` (`src/plus/connection.py::authentify`).
  - **Ответ**: JSON разбор, извлечение `token`, `user`, `account`; обработка `204`/ошибок (`authentify`).
  - **Использование токена**: `Authorization: Bearer <token>` добавляется в `getHeaders()` когда `authorized=True` (`src/plus/connection.py::sendData/getData`).
  - **401 handling**: при `401` выполняется повторная аутентификация и повтор запроса (`sendData`, `getData`).

- **Idempotency (если явно присутствует)**
  - **HTTP заголовок**: `Idempotency-Key` выставляется для **POST** в `sendData()` (`src/plus/connection.py::sendData`).

- **Что именно синхронизируется (как структуры/вызовы)**
  - **Client → Server (push)**
    - **Roast item**: постановка в очередь и отправка на `config.roast_url` (`src/plus/queue.py::queue_roast_item/addRoast`, воркер `Worker.task` отправляет JSON).
    - **Lock schedule**: запрос на `f'{config.lock_schedule_url}?today=YYYY-MM-DD'` с payload `{}` (`src/plus/queue.py::sendLockSchedule`).
    - **Roast properties “sync record”**: в `src/plus/roast.py` перечислены поля `sync_record_*` и фильтрация по whitelist `sync_record_attributes` (`getSyncRecord`).
  - **Server → Client (pull)**
    - **Stock/account state**: `GET config.stock_url?today=YYYY-MM-DD[&lsrt=...]`, JSON обрабатывается `extractAccountState`, `setStock` (`src/plus/stock.py::Worker.fetch`).
    - **Server update для roast по UUID**: `GET f'{config.roast_url}/{uuid}?modified_at=...'`; обработка `204`, `404` (в т.ч. `updateLimitsFromResponse`, `delSync`), `200` → `applyServerUpdates` (`src/plus/sync.py::fetchServerUpdate`).

- **Конфликт/ошибки (если есть явные признаки)**
  - В очереди отправки упомянуты обработки статус‑кодов `401/409/500/429/404` с retry/отбрасыванием (детали механики в анализе не раскрыты).

- **Sync record (roast properties) — известные поля**
  - **Всегда отправляются**: `roast_id`, `location`, `coffee`, `blend`, `amount`, `end_weight`, `defects_weight`, `s_item_id`.
  - **Опциональные (zero-suppressed)**: `density_roasted`, `batch_number`, `batch_pos`, `whole_color`, `ground_color`, `moisture`, `temperature`, `pressure`, `humidity`, `roastersize`, `roasterheating`, `BTU_ELEC`, `BTU_LPG`, `BTU_NG`, `BTU_roast`, `BTU_preheat`, `BTU_bbp`, `BTU_cooling`, `BTU_batch`, `CO2_roast`, `CO2_preheat`, `CO2_bbp`, `CO2_cooling`, `CO2_batch`.
  - **Опциональные (fifty-suppressed)**: `cupping_score`.
  - **Опциональные (empty-string suppressed)**: `label`, `batch_prefix`, `color_system`, `machine`, `notes`, `cupping_notes`.
  - **Whitelist-фильтрация**: `getSyncRecord()` фильтрует по `sync_record_attributes` (лишние поля игнорируются).

## Inferred (ограниченные выводы)

- **Модель синхронизации Artisan+**
  - **Push (client → server)**: через очередь SQLite (`{'url','data','verb'}`) и фонового воркера, который отправляет JSON на `item['url']` (`src/plus/queue.py`).
  - **Pull (server → client)**: периодические `GET` для stock и обновлений roast по `uuid` с параметром `modified_at` (`src/plus/stock.py`, `src/plus/sync.py`).
  - **Bidirectional**: WebSocket‑код существует, но он не привязан фактами из анализа к Artisan+ API; это отдельный механизм транспорта (**DATA GAP** по назначению).

- **Conflict handling**
  - Наличие обработки `409` в воркере очереди указывает, что конфликт как класс ошибок предусмотрен на уровне HTTP‑статуса, но **конкретная стратегия разрешения конфликта** из анализа не следует (**DATA GAP**).

## Data gaps

- **Сущности/контракты**
  - **Roast item payload** (структура `roast_item`, который уходит на `config.roast_url`): **NOT FOUND** в анализе.
  - **Формат server update** (что именно возвращает `GET {roast_url}/{uuid}?modified_at=...` и как устроен `applyServerUpdates`): **NOT FOUND**.
  - **Формат stock/account state** (поля ответа, структура `extractAccountState`/`setStock`): **NOT FOUND**.
  - **Семантика `modified_at`** (тип/формат значения): **NOT FOUND**.

- **API**
  - Точные **HTTP методы** для `config.roast_url`, `config.lock_schedule_url`, `config.stock_url`, `config.roast_url/{uuid}` в анализе **не указаны** (кроме того, что `Idempotency-Key` ставится для POST и что `authentify` — отправка JSON логина).
  - **Полный список endpoint’ов и их схемы запрос/ответ** по `/acoffees`, `/aroast`, `/aschedule/lock`, `/notifications`: присутствуют как константы/фрагменты путей, но payload/ответы **NOT FOUND**.

- **Совместимость**
  - Требования сервера к `Content-Encoding: gzip`/поддержке сжатия: в клиенте это есть, но серверная сторона не описана (**DATA GAP**, но клиент будет так отправлять).
  - Реальная логика retry/backoff/“drop” в очереди для `409/500/429/404`: в анализе нет алгоритма (**DATA GAP**).

## Recommendations (RECOMMENDATION)

- **RECOMMENDATION (совместимость MUST)**: серверная сторона должна принимать `Authorization: Bearer <token>`, корректно отвечать `401` при истечении/невалидности токена (клиент выполнит re-auth и повтор).
- **RECOMMENDATION (совместимость MUST)**: сервер должен принимать `Content-Type: application/json; charset=utf-8` и **поддерживать `Content-Encoding: gzip`**, т.к. клиент может отправлять сжатый JSON при превышении порога.
- **RECOMMENDATION (совместимость MUST)**: для POST‑операций, где клиент ставит `Idempotency-Key`, сервер должен поддерживать идемпотентность на этом ключе (иначе возможны дубликаты при ретраях/повторах).
- **RECOMMENDATION (MVP можно опустить)**: если MVP — только Artisan+ sync, можно не включать WebSocket‑механизмы (`src/artisanlib/wsport.py`, `src/artisanlib/weblcds.py`) и прочие не‑plus HTTP интеграции (Shelly/GitHub/roastlog), т.к. они не требуются фактами для описанного HTTPS sync‑потока.

## Черновик API (таблица endpoint’ов) — только то, что видно в анализе

| Endpoint | Direction | Method | Назначение (по фактам) | Auth | Idempotency | Payload/Response |
|---|---|---|---|---|---|---|
| `config.auth_url` (напр. `https://artisan.plus/api/v1/accounts/users/authenticate`) | client → server | **DATA GAP** (в коде “send”) | Аутентификация, получение `token` | без Bearer (`authorized=False`) | **DATA GAP** | request JSON: `email`, `password`; response: JSON с `token`, `user`, `account` |
| `config.stock_url?today=YYYY-MM-DD[&lsrt=...]` | server → client | **DATA GAP** (в коде `getData`) | Получение account/stock state | Bearer | n/a | response JSON: **DATA GAP** |
| `config.roast_url` | client → server | **DATA GAP** (очередь хранит `verb`) | Отправка `roast_item` | Bearer | возможно (если POST) | request JSON: **DATA GAP** |
| `f'{config.roast_url}/{uuid}?modified_at=...'` | server → client | **DATA GAP** | Получение обновлений roast по UUID | Bearer | n/a | response: `204/404/200`; тело для `200/404`: **DATA GAP** |
| `f'{config.lock_schedule_url}?today=YYYY-MM-DD'` | client → server | **DATA GAP** | Lock schedule | Bearer | **DATA GAP** | request JSON: `{}`; response: **DATA GAP** |

## Матрица синхронизации (entity × direction)

| Entity | client → server | server → client | Mode |
|---|---:|---:|---|
| Credentials (`email`, `password`) | Yes | No | push (auth request) |
| Auth token (`token`) | No | Yes | pull (auth response) |
| Roast item (`roast_item`) | Yes | **DATA GAP** | push |
| Roast server update (by `uuid`, `modified_at`) | **DATA GAP** | Yes | pull |
| Stock/account state | No | Yes | pull |
| Schedule lock | Yes | No | push |
| Roast properties sync record (`sync_record_*` поля) | Yes | **DATA GAP** | push |

## Совместимость vs MVP

- **Обязательно для совместимости (из фактов/неизбежностей клиента)**
  - Bearer‑авторизация на защищённых запросах (`Authorization: Bearer ...`).
  - Поведение на `401` (клиент делает re-auth и повторяет запрос).
  - Приём JSON и, потенциально, gzip‑сжатого JSON (`Content-Encoding: gzip`).
  - Учёт `Idempotency-Key` для POST (клиент его ставит).

- **Можно опустить в MVP (RECOMMENDATION, если цель — только описанный плюс‑sync)**
  - WebSocket‑канал и связанные обработчики.
  - Не‑plus интеграции (Shelly устройства, GitHub update check, roastlog загрузки), т.к. они не являются частью обнаруженного `src/plus/*` sync‑контракта.

