# Интеграция Artisan с CoffeeWave Plus (анализ по HAR)

Документ объединяет анализ репозитория клиента Artisan для подготовки интеграции с внешним веб-сервисом CoffeeWave Plus, контракт которого известен по HAR.

---

## Контекст API (из HAR)

### Auth: Laravel Sanctum SPA-cookie auth

**Flow:**

1. `GET https://plus.coffeewave.me/sanctum/csrf-cookie` → cookies `XSRF-TOKEN` + session  
2. `POST https://plus.coffeewave.me/api/login` (JSON `email`/`password`/`remember`) + header `X-XSRF-TOKEN` (url-decoded значение cookie)  
3. Далее запросы идут с cookies + `X-XSRF-TOKEN` (при state-changing запросах).

### Roasts import варианты

**A) POST /api/roasts/upload** (multipart/form-data, поле `file`=.alog) — автоимпорт, но может вернуть 422:

- «Обжарка с данным профилем уже существует» (дубликат по UUID/профилю)
- «В профиле не указана локация» (если в .alog нет location)

**B) Надёжный путь:**

1. `POST /api/roasts` (application/json) — создать roast карточку. Валидации минимум:
   - `bean_id` обязателен
   - `ingredients` обязателен, включая `ingredients[0].bean_lot_id`
   - `location_id` / `machine_id` / `created_by_id` (в рабочем сценарии присутствуют)
2. `POST /api/roasts/{id}/upload` (multipart, поле `file`=.alog) — прикрепить профиль, после чего `profile_available` становится true.

### Другие сущности в API

`/api/beans`, `/api/lots`, `/api/locations`, `/api/machines`, `/api/references`, `/api/me/settings` и т.д.

---

## 1. Где в коде лежат/формируются данные roast и .alog

### Roast UUID (idempotency)

- **Где хранится:** `ArtisanQGraphicsScene.roastUUID` (`src/artisanlib/canvas.py`, ~строка 2040), дублируется в `profile['roastUUID']`.
- **Где задаётся:**
  - При загрузке: `src/artisanlib/main.py:16073-16077` — из `profile['roastUUID']` или `uuid.uuid4().hex`.
  - При сохранении: `main.py:17000-17004` — если `roastUUID is None`, генерируется и пишется в `profile['roastUUID']`.
  - При DROP: `canvas.py:18817-18820` — если ещё нет, генерируется.
  - При «Save Copy»: `main.py:17232-17235` — новый `uuid.uuid4().hex` в копии.
- **Участвующие поля:** только `roastUUID` (строка, 32 hex-символа). Регистрация пути: `plus.register.addPath(uuid, path)` при сохранении/загрузке.

### Time-series (BT/ET/Δ, extra), events, time_marks

- **Внутри профиля (qmc):** `canvas.py`: `timex`, `temp1` (ET), `temp2` (BT), `delta1`/`delta2`, `extratimex`, `extratemp1`, `extratemp2`; `specialevents`, `specialeventstype`, `specialeventsvalue`, `timeindex` (индексы CHARGE, DRY, FCs, FCe, DROP, COOL и т.д.).
- **В словаре профиля (ProfileData):** `src/artisanlib/atypes.py:204-206,195-198` — `timex`, `temp1`, `temp2`, `specialevents`, `specialeventstype`, `specialeventsvalue`, `timeindex`.
- **Где формируются при сохранении:** `main.py:17015-17018` — `profile['timex']`, `profile['temp1']`, `profile['temp2']`, `profile['specialevents']` и т.д. из `self.qmc.*`.

### Экспорт/импорт .alog

- **Сериализация:** `src/artisanlib/util.py:971-975` — `serialize(filename, obj)`: запись `repr(obj)` в файл (текст, Python-представление dict).
- **Десериализация:** `src/artisanlib/util.py:978-987` — `deserialize(filename)`: `ast.literal_eval(f.read())`.
- **Расширение:** по умолчанию `.alog` (`plus/config.py:34`, диалоги в `main.py`).
- **Где вызывается запись:** `main.py:13208,17246` — после `getProfile()` и `plusAddPath()` вызывается `serialize(filename_path, pf)` (autosave и обычный save).
- **Метаданные в .alog:** всё из `ProfileData` (в т.ч. `roastUUID`, `title`, `beans`, `weight`, `roastertype`, `machinesetup`, `roastdate`, `roastepoch`, `plus_store`, `plus_coffee`, `plus_blend_spec`, события, timex/temp1/temp2, extra-каналы и т.д.). Отдельного поля **location** (как географическая/складская локация для API) в профиле нет — есть только **plus_store** (store/location hr_id для artisan.plus).

### «Location» в клиенте

- **Где хранится:** в контексте artisan.plus — `qmc.plus_store` (hr_id склада/локации), `qmc.plus_store_label`. Это не отдельная сущность «location» в .alog, а выбор склада для кофе/бленда.
- **В roast/sync:** `src/plus/roast.py:363-365` — `d['location'] = aw.qmc.plus_store` или `None`; при отсутствии кофе и бленда location обнуляется (стр. 393-396).
- **В .alog:** в профиле есть `plus_store` / `plus_store_label`, но **нет** отдельного поля типа `location_id` для внешнего API (как в CoffeeWave).

---

## 2. Сетевая инфраструктура и точки расширения

### HTTP

- **Модуль:** `src/plus/connection.py`. Используется `requests`; только JSON (POST/PUT/GET), без multipart.
- **Методы:** `sendData(url, data, verb='POST'|'PUT')`, `getData(url, ...)`; заголовки через `getHeaders(authorized)`; Idempotency-Key для POST; Bearer token из `config.token`.
- **Auth:** artisan.plus — логин/пароль → токен, без Sanctum CSRF/cookies.

### Концепция «отправить профиль куда-то»

- **Очередь отправки:** `src/plus/queue.py` — `addRoast()` формирует payload через `roast.getRoast()` и кладёт в очередь `{'url': roast_url, 'data': roast_item, 'verb': 'POST'}`. Воркер вызывает `connection.sendData(...)`. Файл .alog никуда не отправляется.
- **Триггеры:** (1) DROP при включённом plus и авто-DROP (`canvas.py:6211,18888-18889`); (2) ручной sync по клику на иконку Plus (`controller.toggle` → `queue.addRoast(unsynced=True)`); (3) сохранение профиля → при изменении sync record может ставиться задача на обновление (`controller.updateSyncRecordHashAndSync` → при необходимости `addRoast(sync_record)` в `controller.py:443,451`); (4) Roast Properties → кнопка, вызывающая `plus.queue.addRoast()` (`roast_properties.py:5711`).
- **Плагинов/exporters/hooks on-save/on-stop в явном виде нет** — логика зашита в plus (queue, controller, sync).

### Где лучше встраивать CoffeeWave Plus

- **Вариант A:** отдельный модуль (например `coffeewave_plus/` или `plus/coffeewave.py`), свой клиент (Sanctum cookies + multipart), вызов из тех же точек, что и plus: после сохранения .alog и/или после DROP (по настройке или второму «сервису»).
- **Вариант B:** расширить текущий `plus` вторым бэкендом (artisan.plus vs CoffeeWave Plus) с общим интерфейсом «upload roast / upload .alog» и разной реализацией auth и запросов.

Чистая точка расширения: **после успешного `serialize(..., pf)` в `main.py` (файл сохранён)** и **при вызове `addRoast()` (DROP или ручной sync)** — оттуда вызывать новый клиент (создание roast + загрузка .alog при необходимости).

---

## 3. Сопоставление модели клиента с API CoffeeWave Plus

### Что клиент может дать для POST /api/roasts (по текущему коду)

| Поле API      | В клиенте   | Источник |
|---------------|-------------|----------|
| label         | title       | `qmc.title` → `profile['title']` |
| amount        | start_weight (kg) | `weight[0]`, конвертация в kg в `roast.getTemplate` |
| end_weight    | weight[1] в kg | есть в getTemplate → end_weight |
| bean_id       | **Нет**     | Нет справочника «beans» с id; есть строка `beans` и plus_coffee (hr_id кофе) |
| ingredients / bean_lot_id | **Нет** | Есть plus_blend_spec (кофе + ratio), нет lot_id |
| machine_id    | **Нет**     | Есть roastertype/machinesetup (строка), нет числового machine_id |
| location_id   | **Нет**     | Есть plus_store (hr_id склада artisan.plus), не location_id CoffeeWave |
| reference_id  | Нет в текущем roast-модели | — |
| roast_id (UUID) | roastUUID | `qmc.roastUUID` |

Для **POST /api/roasts/upload** (multipart, file=.alog) серверу нужна в т.ч. **location в профиле** — в .alog хранится только `plus_store` (и то только при использовании artisan.plus), отдельного поля «location» для CoffeeWave в .alog нет.

### Чего не хватает относительно HAR API

- **Обязательные для создания roast:** `bean_id`, `ingredients` (в т.ч. `ingredients[0].bean_lot_id`), в рабочем сценарии — `location_id`, `machine_id`, `created_by_id`.
- **Для автоимпорта (POST /api/roasts/upload):** в .alog должна быть **location** (в профиле её нет в виде, ожидаемом CoffeeWave).
- **Справочники:** в клиенте нет загрузки/выбора `/api/beans`, `/api/lots`, `/api/locations`, `/api/machines` — только свой stock (acoffees) и плюс-склады.

### Стратегия маппинга

- **bean_id / bean_lot_id:** справочников beans/lots в клиенте нет. Варианты: (1) ввод вручную или выбор из загруженного списка (GET /api/beans, /api/lots) в отдельном диалоге/полях; (2) маппинг по имени кофе (plus_coffee_label / beans) к bean_id с риском неоднозначности; (3) не поддерживать надёжный путь B без ввода/выбора bean и lot.
- **location_id / machine_id:** либо отдельные справочники (GET /api/locations, /api/machines) с выбором в UI, либо конфиг/настройки (дефолтные location_id и machine_id для учёта), либо оба варианта.

---

## 4. Выходные артефакты

### A) Таблица «Что нужно для интеграции → где в коде → как извлечь/сформировать»

| Что нужно | Где в коде | Как извлечь/сформировать |
|-----------|------------|---------------------------|
| Roast UUID (idempotency) | `qmc.roastUUID` (`canvas.py:2040`), `getProfile()['roastUUID']` (`main.py:17004`) | Уже есть; при создании roast на CW — можно отправить как внешний id или не передавать, если CW генерирует свой. |
| Файл .alog для upload | Путь: `aw.curFile` при сохранении; содержимое — результат `getProfile()` + `serialize()` | После `serialize(path, pf)` использовать `path` и читать файл для multipart или передать path в клиент. |
| Time-series (BT/ET/Δ/extra) | `profile['timex']`, `profile['temp1']`, `profile['temp2']`, `profile['extratimex']`, … | Из `aw.getProfile()` или из уже сериализованного .alog (deserialize). |
| Events / time_marks | `profile['specialevents']`, `profile['specialeventstype']`, `profile['specialeventsvalue']`, `profile['timeindex']` | Из `getProfile()` или deserialize(.alog). |
| Метаданные (label, amount, date, machine string) | `plus/roast.py: getTemplate()`, `getRoast()`; `main.getProfile()` | `roast.getRoast()` даёт текущий «sync record»; для CW нужно маппить в bean_id, ingredients, location_id, machine_id. |
| Location (для API) | В клиенте только `qmc.plus_store` (плюс-склад) | Нет location_id. Нужен справочник /api/locations или конфиг; при автоимпорте — доп. поле в профиле или настройка «default location_id». |
| Bean / lot | `qmc.beans`, `qmc.plus_coffee`, `qmc.plus_blend_spec` | Нет bean_id/bean_lot_id. Нужны справочники или ручной ввод. |

### B) Точки вставки (hook/handler)

| Точка | Файл | Зачем |
|-------|------|--------|
| После сохранения .alog | `main.py` — в `filesave()` после `serialize(..., pf)` и `plusAddPath(...)` (~17245–17247) | Вызвать CoffeeWave Plus: авто-upload .alog и/или создание roast + привязка файла. |
| После DROP (очередь roast) | `canvas.py` — рядом с `addRoast()` (~6222, ~18899) | По настройке дублировать отправку в CW Plus (create roast + upload .alog) или только upload. |
| Ручной «Upload to CoffeeWave» | Меню или кнопка в Roast Properties / File | Вызов клиента: get_csrf_cookie → login → create_roast (если нужен надёжный путь) → upload_alog. |

### C) Черновик интерфейса модуля CoffeeWave Plus

```python
# Модуль: coffeewave_plus/client.py (или plus/coffeewave_client.py)

# Требования: session с cookie jar, отправка X-XSRF-TOKEN (url-decoded значение cookie XSRF-TOKEN),
# логирование запросов (без тел с паролями), обработка 422 с телом ошибки (дубликат, «локация не указана»).

class CoffeeWavePlusClient:
    def __init__(self, base_url: str, session: requests.Session | None = None):
        # session с cookie jar; при state-changing запросах добавлять X-XSRF-TOKEN из cookies.

    def get_csrf_cookie(self) -> bool:
        # GET {base}/sanctum/csrf-cookie → сохранить cookies (XSRF-TOKEN, session).

    def login(self, email: str, password: str, remember: bool = False) -> bool:
        # После get_csrf_cookie: POST {base}/api/login, JSON {email, password, remember},
        # header X-XSRF-TOKEN = urllib.parse.unquote(cookies['XSRF-TOKEN']).

    def create_roast(self, payload: dict) -> requests.Response:
        # POST {base}/api/roasts, application/json. payload должен содержать минимум bean_id, ingredients
        # (в т.ч. ingredients[0].bean_lot_id), опционально location_id, machine_id, created_by_id, label, amount, end_weight, reference_id.
        # Обработка 422, логирование.

    def upload_alog(self, roast_id: int | str, file_path: str | bytes) -> requests.Response:
        # POST {base}/api/roasts/{roast_id}/upload, multipart/form-data, поле "file" = .alog файл.
        # После успеха profile_available = true на сервере.

    def upload_alog_auto(self, file_path: str) -> requests.Response:
        # POST {base}/api/roasts/upload, multipart/form-data, поле "file" = .alog.
        # Может вернуть 422: «Обжарка с данным профилем уже существует» или «В профиле не указана локация».
```

Входы: для `create_roast` — bean_id, ingredients (с bean_lot_id), location_id, machine_id (и при необходимости created_by_id, label, amount, end_weight); для upload — путь к .alog или его содержимое. Логировать URL и заголовки, не логировать пароли и полные тела с кредами.

---

## 5. Результаты ripgrep по ключевым словам

| Ключ | Файл:строка | Что происходит |
|------|-------------|----------------|
| **alog** | `main.py:355,398,403` | Упоминание .alog при открытии профиля/шаблона по URL и при двойном клике. |
| **alog** | `main.py:12928,13229,13473,13481` | Сохранение/диалоги: по умолчанию `*.alog`, проверка расширения. |
| **alog** | `config.py:34` | `profile_ext: Final[str] = 'alog'`. |
| **uuid / UUID** | `main.py:16076-16077,17001-17004,17234-17235` | Генерация `roastUUID`: `uuid.uuid4().hex` при загрузке и в `getProfile()`. |
| **uuid** | `canvas.py:18817-18820` | Генерация UUID при первом DROP (если ещё нет). |
| **uuid** | `canvas.py:2039-2040` | Атрибут `self.roastUUID: str \| None`. |
| **uuid** | `plus/config.py:35` | `uuid_tag = 'roastUUID'` (в .alog и как roast_id на сервере). |
| **upload** | `canvas.py:6211,18888-18889` | После DROP при `plus_account` вызывается `addRoast()`. |
| **http/requests** | `plus/connection.py:37-38,397-406,454` | Основной HTTP: `requests.post`/`put`/`get`, Bearer token. |
| **token** | `plus/connection.py:68-106,358-359` | `getToken/setToken`, заголовок `Authorization: Bearer {token}`. |
| **sync** | `main.py:717,13692,13201-13206` | Импорт `plus.sync`, вызовы `sync.sync()`, `updateSyncRecordHashAndSync`. |
| **profile** | `main.py:16911+` | `getProfile()` собирает словарь профиля из `self.qmc.*` и др. |
| **location** | `plus/roast.py:363-365,393` | `d['location'] = aw.qmc.plus_store` или `None`. |
| **machine** | `plus/roast.py:153-154` | В шаблон: `roastertype`→`machine`, `machinesetup`→`setup`. |
| **bean** | `atypes.py:154` | `beans: str` в ProfileData. |

---

## 6. Missing client features vs HAR API (CoffeeWave Plus)

Чеклист недостающего функционала клиента относительно контракта CoffeeWave Plus по HAR.

- [ ] **Sanctum-auth клиентская реализация:** GET `/sanctum/csrf-cookie` + cookie-jar + `X-XSRF-TOKEN` для state-changing запросов.
- [ ] **Импорт roast по надёжному сценарию:** `POST /api/roasts` (JSON с `bean_id`, `ingredients[].bean_lot_id`, `location_id`, `machine_id`, `label`, `amount`…) → затем `POST /api/roasts/{id}/upload` (multipart `file=.alog`).
- [ ] **Авто-импорт** `POST /api/roasts/upload` (multipart), включая обработку 422:
  - дубликат по roastUUID/профилю,
  - отсутствие location в `.alog`.
- [ ] **Источник/маппинг `location_id` и `machine_id`** (локальная конфигурация или загрузка справочников `/api/locations`, `/api/machines`).
- [ ] **Источник/маппинг `bean_id` и `bean_lot_id`** (инвентарь на стороне клиента или UI-маппинг к серверному inventory).
- [ ] **Логирование интеграции** (запрос/ответ/ошибка) + режим «dry-run» без загрузки на сервер.
- [ ] **Обратная синхронизация (опционально):** поиск roast по UUID `GET /api/roasts/uuid/{uuid}`, скачивание `.alog` `GET /api/roasts/{id}/download`, экспорт xlsx `/api/file/roasts/export`.
- [ ] **UI/настройки клиента для сервера** (base URL, креды/токены, default location/machine, политика идемпотентности).

### Привязка к коду (кратко)

- **Sanctum-auth** — в `plus/connection.py` сейчас только Bearer и `sendData`/`getData` с JSON. Нужен отдельный слой: сессия с cookie jar, GET csrf-cookie, POST login с телом и заголовком `X-XSRF-TOKEN` из cookie (url-decode).
- **Надёжный импорт** — встроить рядом с вызовами `plus.queue.addRoast()` и после `serialize()` в `main.py` (~17245–17247). Отдельный поток: собрать payload (bean_id, ingredients, location_id, machine_id, label, amount), POST `/api/roasts`, затем multipart POST `/api/roasts/{id}/upload` с .alog.
- **Авто-импорт** — после сохранения .alog в том же месте в `filesave()` вызывать multipart upload; обрабатывать 422 (дубликат / нет location).
- **location_id / machine_id** — в коде только `qmc.plus_store` и строки `roastertype`/`machinesetup`. Варианты: QSettings (default location_id/machine_id) и/или загрузка списков из `/api/locations`, `/api/machines` с выбором в диалоге.
- **bean_id / bean_lot_id** — справочников beans/lots в клиенте нет; есть `plus_coffee`, `beans`, `plus_blend_spec`. Нужен диалог/поля с загрузкой из `/api/beans`, `/api/lots` или конфиг/маппинг.
- **Логирование и dry-run** — отдельный logger (например `logging.getLogger('coffeewave_plus')`) и флаг dry_run: при True не выполнять POST upload/POST roasts, только логировать payload и заголовки.
- **Обратная синхронизация** — новые вызовы GET по UUID, GET download .alog, GET export xlsx; точки входа: меню/кнопка «Скачать с CoffeeWave» и при открытии по ссылке (аналог `artisan://roast/<UUID>`).
- **UI/настройки** — отдельная секция в настройках (base URL, логин/пароль или токен, default location_id/machine_id, «включить авто-загрузку», «dry-run»). Хранить в QSettings или в отдельном конфиге интеграции.

---

## Ограничения безопасности

В отчёте и в коде интеграции не должны фигурировать реальные токены, куки, пароли и URL с кредами. При попадании в отчёт — заменять на «REDACTED».
