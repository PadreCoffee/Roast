# END-OF-TASK REPORT: Roast-like hover, bubble, debug

## Что сделано

### 1. Расширенный debug и переменные окружения
- **ROAST_DEBUG=1** — включает debug (логи и артефакты).
- **ROAST_DEBUG_DIR** (если задан) — все артефакты пишутся в эту директорию:
  - `debugging.log` (один файл, truncate при старте сессии)
  - `roast_like_render.png`, `roest_hover_snapshot.png`, `hover_state.json`
- Если **ROAST_DEBUG_DIR** не задан — используется по умолчанию:
  - macOS: `~/Library/Logs/Roast`
  - иначе: `~/.roast/logs`
  - repo-копия лога: `debugging.log` в корне репозитория (при выходе копируется из системного лога; при заданном ROAST_DEBUG_DIR копирование не делается).

### 2. Снапшоты при ROAST_DEBUG=1
- **roast_like_render.png** — в конце каждого redraw в roast-like режиме (в ROAST_DEBUG_DIR или в дефолтную директорию логов).
- **roest_hover_snapshot.png** — при первом hover после открытия профиля (в ту же директорию).

### 3. Lifecycle artists для hover
- Вертикальные линии:
  - **roest_vline_roast** (`self.roest_vline_roast`) — Line2D на `ax` (основной график).
  - **roest_vline_controls** (`self.roest_vline_controls`) — Line2D на `ax_controls`.
  - Внутри по-прежнему используются `_roast_hover_line` и `_roast_hover_line_controls`; алиасы выставляются в `_roast_hover_ensure_artists()`.
- Маркеры:
  - Хранятся в **dict** `_roast_hover_markers_dict` по ключам кривых (`roast_et`, `roast_bt`, `roast_ror_et`, `roast_ror_bt`, `extra_N`, `control_0` …).
  - На hover обновляются через **set_data** и **set_color**, без пересоздания; неиспользуемые ключи скрываются через **set_visible(False)**.

### 4. Вычисление Y для кривых
- **Непрерывные (BT/ET/extra, RoR):** интерполяция по времени — `_line_value_at(..., use_interp=True)` (numpy.interp).
- **RoR:** маркер в координатах `delta_ax` (transform линии), отображается на той же оси.
- **Controls (step):** «последнее значение в момент ≤ x» — через `_control_line_value_at()` (внутри `_step_value_at(E?timex, E?values, x)` для E1–E4, иначе `_line_value_at` по данным линии).

### 5. Позиционирование bubble
- Один bubble — одна **Annotation** (`_roast_hover_anno`), без второго «фиксированного» тултипа.
- **xy** обновляется на каждом hover: `(x_cursor, y_cursor)` = `(t_sec, event.ydata)` для активной оси (`event.inaxes`).
- **Clamp:** текст тултипа (xytext) смещён относительно точки и ограничен пределами оси (xlim/ylim).
- При смене оси (ax / delta_ax / ax_controls) аннотация переезжает на новую ось (`remove` + `add_artist`).
- Старый фиксированный HUD (Text + Rectangle в углу) при отображении bubble не показывается (всегда скрыт).

### 6. Обязательные логи в debugging.log
- **При каждом redraw (roast-like):** одна строка NDJSON с полями:
  - `event`: `redraw_roast_like`
  - `vline_roast`, `vline_roast_visible`
  - `vline_controls`, `vline_controls_visible`
  - `bubble_artist`
  - `marker_count_roast`, `marker_count_controls`
  - `x_cursor`: `null` (на redraw курсора нет)
- **При первом hover:** одна строка NDJSON с полями:
  - `event`: `first_hover`
  - `x_cursor`: t_sec
  - `inaxes`, `inaxes_id`, `active_axis`
  - `control_series_updated`: количество обновлённых control-серий (маркеров на ax_controls).

---

## Как воспроизвести

1. Включить debug:
   ```bash
   export ROAST_DEBUG=1
   ```
   Опционально — задать свою папку для артефактов:
   ```bash
   export ROAST_DEBUG_DIR=/path/to/my/debug/artifacts
   ```

2. Запустить приложение и открыть профиль (roast-like режим).

3. Дождаться полного redraw графика (должен появиться/обновиться `roast_like_render.png` в выбранной директории).

4. Провести курсором над графиком (основная ось, RoR, панель контролов) — первый hover пишет `roest_hover_snapshot.png` и строку `first_hover` в лог.

5. Продолжить двигать курсор — bubble должен следовать за курсором, вертикальная линия по x = t_sec, маркеры на кривых и контролах на месте.

6. Проверить classic-режим: переключить обратно в classic, убедиться, что crosshair/поведение не сломаны.

---

## Что смотреть в debugging.log

- **Строки `"event": "redraw_roast_like"`:**
  - `vline_roast` / `vline_controls` — должны быть `true` после первого redraw с данными.
  - `vline_roast_visible` / `vline_controls_visible` — на redraw могут быть `false` (линии показываются при hover).
  - `bubble_artist` — `true` после ensure_artists.
  - `marker_count_roast` / `marker_count_controls` — на redraw обычно 0 (маркеры появляются при hover).

- **Строка `"event": "first_hover"`:**
  - `x_cursor` — время под курсором (секунды).
  - `active_axis` — в какой оси был курсор: `ax`, `delta_ax`, `ax_controls` или `ax_phases`.
  - `control_series_updated` — число control-серий, для которых посчитан маркер (должно совпадать с числом видимых E1–E4 линий при наличии данных).

- **Строки `"event": "hover_tooltip"` (каждые ~50 hover):**
  - `x` — t_sec; `included_series`, `controls_count`, `parts_count` — для проверки состава bubble.

- **Ошибки:** строки с `"event": "RENDER_SNAPSHOT_ERROR"`, `"roest_hover_snapshot_error"`, `"hover_state_json_error"` — указывают на сбой записи артефактов или состояния.

Файл в формате NDJSON (одна JSON-строка на строку файла); при ROAST_DEBUG_DIR заданном весь лог и снапшоты лежат в одной директории.
