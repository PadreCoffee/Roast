# Network Facts From Code

| location (file + symbol) | direction | protocol | endpoint / topic / address | request payload (если виден) | response handling (если есть) | auth details |
|---|---|---|---|---|---|---|
| `src/plus/config.py::api_base_url` (и соседние константы) | send | HTTPS | `https://artisan.plus/api/v1`, `https://artisan.plus`, `https://artisan.plus/api/v1/accounts/users/authenticate`, `/acoffees`, `/aroast`, `/aschedule/lock`, `/notifications` | NOT FOUND | NOT FOUND | NOT FOUND |
| `src/plus/connection.py::authentify` | send | HTTPS | `config.auth_url` | JSON: `{'email': aw.plus_account, 'password': config.passwd}` | JSON разбор ответа, извлечение `token`, `user`, `account`, обработка 204/JSON/ошибок | Нет Authorization (вызов `sendData(..., authorized=False)`) |
| `src/plus/connection.py::sendData` | send | HTTPS | `url` (входной параметр) | JSON сериализация `data`, `Content-Type: application/json; charset=utf-8`, gzip при `compress` и `post_compression_threshold`, `Idempotency-Key` для POST | При `401` повторная аутентификация и повтор запроса | `Authorization: Bearer <token>` в `getHeaders()` при `authorized=True` |
| `src/plus/connection.py::getData` | receive | HTTPS | `url` + `params` | query params через `params` | При `401` повторная аутентификация и повтор запроса | `Authorization: Bearer <token>` в `getHeaders()` при `authorized=True` |
| `src/plus/stock.py::Worker.fetch` | receive | HTTPS | `config.stock_url?today=YYYY-MM-DD[&lsrt=...]` | NOT FOUND | JSON разбор: `extractAccountState`, `setStock`, обработка `204` | Через `connection.getData()` (Bearer в `getHeaders`) |
| `src/plus/sync.py::fetchServerUpdate` | receive | HTTPS | `f'{config.roast_url}/{uuid}?modified_at=...'` | NOT FOUND | Обработка `204`, `404` (JSON разбор, `updateLimitsFromResponse`, `delSync`), `200` (JSON, `applyServerUpdates`) | Через `connection.getData()` (Bearer в `getHeaders`) |
| `src/plus/queue.py::Worker.task` | send | HTTPS | `item['url']` из очереди | Очередь SQLite: `{'url', 'data', 'verb'}`; `data` отправляется как JSON в `sendData` | `raise_for_status`, разбор JSON при ответе; статус‑коды 401/409/500/429/404 с retry/отбрасыванием | `sendData` использует Bearer при `authorized=True` |
| `src/plus/queue.py::queue_roast_item` / `addRoast` | send | HTTPS | `config.roast_url` | `roast_item` JSON (через `sendData` в воркере) | NOT FOUND (обработка в `Worker.task`) | Bearer через `sendData` |
| `src/plus/queue.py::sendLockSchedule` | send | HTTPS | `f'{config.lock_schedule_url}?today=YYYY-MM-DD'` | `{}` | NOT FOUND (обработка в `Worker.task`) | Bearer через `sendData` |
| `src/artisanlib/wsport.py::wsport.connect` | bidirectional | WebSocket (`ws://`) | `ws://{host[:port]}/{path}` | Текстовые сообщения; см. `send()` | Входящие сообщения: `json.loads`, обработка `pushMessage` и ответов по `id` | NOT FOUND |
| `src/artisanlib/wsport.py::wsport.send` | send/receive | WebSocket (`ws://`) | `ws://{host[:port]}/{path}` | JSON сериализация `request` с `id` и `roasterID` | Ожидание ответа по `id`, таймаут `request_timeout` | NOT FOUND |
| `src/artisanlib/weblcds.py::WebView.startup` + `WebView.websocket_handler` | bidirectional | WebSocket (aiohttp) | `0.0.0.0:{port}` + `/{websocket_path}` | Текстовые сообщения | Если клиент шлёт пустую строку — отправляется последний message | NOT FOUND |
| `src/artisanlib/comm.py::Shelly3EMPro_EnergyReturn` | receive | HTTP | `http://{shelly_3EMPro_host}/rpc/EMData.GetStatus?id=0` | NOT FOUND | `r.json()` и чтение `total_act`, `total_act_ret` | NOT FOUND |
| `src/artisanlib/comm.py::Shelly3EMPro_ActivePower_ApparentPower` | receive | HTTP | `http://{shelly_3EMPro_host}/rpc/EM.GetStatus?id=0` | NOT FOUND | `r.json()` и чтение `total_act_power`, `total_aprt_power` | NOT FOUND |
| `src/artisanlib/comm.py::updateShellyPlusPlug` | receive | HTTP | `http://{shelly_PlusPlug_host}/rpc/Switch.GetStatus?id=0` | NOT FOUND | `r.json()` и чтение `aenergy`, `apower`, `temperature` | NOT FOUND |
| `src/artisanlib/main.py::shellyrelay` (ветка `c.startswith('shellyrelay')`) | send | HTTP | `http://{shelly_PlusPlug_host}/relay/{n}?turn=on|off` | NOT FOUND | `raise_for_status()` | NOT FOUND |
| `src/artisanlib/main.py::artisanURLextractor` | receive | HTTP/HTTPS | `url.toString()` | NOT FOUND | `r.text` парсится через `ast.literal_eval` | NOT FOUND |
| `src/artisanlib/main.py::checkUpdate` | receive | HTTPS | `https://api.github.com/repos/artisan-roaster-scope/artisan/releases/latest` | NOT FOUND | JSON разбор `tag_name`, сравнение версий | NOT FOUND |
| `src/artisanlib/roastlog.py::extractProfileRoastLog` | receive | HTTPS | `url.toString()` и `https://roastlog.com/roasts/profiles/?rid=...` | NOT FOUND | HTML разбор `page.content`; затем `response.json()` | NOT FOUND |
| `src/artisanlib/modbusport.py::modbusport.connect_async` | bidirectional | MODBUS TCP/UDP | `host`, `port` (TCP/UDP) | NOT FOUND | `await self._client.connect()`; обновление регистров | NOT FOUND |
| `src/artisanlib/util.py::isOpen` | send | raw TCP | `(ip, port)` | NOT FOUND | `socket.connect_ex(...) == 0` | NOT FOUND |
| `src/artisanlib/devices.py::DeviceTable.getTaskURL` | send | HTTP (URL формирование) | `http://{hostname}.local:{port}/{index_path}` | NOT FOUND | NOT FOUND | NOT FOUND |

## NOT FOUND
- SSE (Server-Sent Events) implementation
- MQTT
- AMQP / RabbitMQ
- gRPC
- raw UDP sockets (не MODBUS)

## UNCERTAIN
- None
