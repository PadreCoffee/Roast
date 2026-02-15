# Отчёт по правкам: Roast-like layout и Hover bubble

Документ описывает две группы изменений в `src/artisanlib/canvas.py`: настройка осей roast-like (время по X, controls, phases) и рефакторинг построения строк hover bubble (канонический порядок, BG-пары, bg-only режим).

---

## 1. Roast-like layout: оси, тики, метки времени

### Цель

- Метки времени по оси X — под **roast** (main chart), а не под controls; phases bar — без подписей X.
- Ось Y у controls: только тики 0, 25, 50, 75, 100; без minor ticks и minor grid; «воздух» по Y: -10…110.
- Уменьшить зазор между roast и controls (hspace и высота блока controls).
- Не ломать hover / vline / bubble.

### 1.1 Тики и сетка ax_controls

**Где:** `redraw_roast_like()` — блок настройки `ax_controls` (два пути: первое создание осей и путь `forceRenewAxis`).

**Было:**

- `ax_controls.set_ylim(0, 100)`
- `yaxis.set_major_locator(ticker.MultipleLocator(10))`, `set_minor_locator(ticker.MultipleLocator(5))`
- Два вызова `grid(..., which='major', axis='y', ...)` и `grid(..., which='minor', axis='y', ...)`

**Стало:**

- `ax_controls.set_ylim(-10, 110)` — запас по вертикали.
- `ax_controls.set_yticks([0, 25, 50, 75, 100])` — только эти тики.
- `ax_controls.yaxis.set_minor_locator(ticker.NullLocator())` — minor отключены.
- Оставлен только **major** grid по Y; вызов `grid(True, which='minor', axis='y', ...)` удалён.

### 1.2 Ось X: где показываются метки времени

**Где:** тот же `redraw_roast_like()` (оба пути) и блок с `show_roast_time_labels` в пути `forceRenewAxis`.

**Изменения:**

| Ось | Было | Стало |
|-----|------|--------|
| **self.ax** (roast) | `tick_params(axis='x', labelbottom=False)` в первом блоке; в forceRenewAxis: `show_roast_time_labels = not self.roast_layout_like` → при roast метки выключены | `tick_params(axis='x', labelbottom=True)` в первом блоке; в forceRenewAxis: `show_roast_time_labels = self.roast_layout_like` → при roast метки **включены** под roast |
| **self.ax_controls** | Уже `labelbottom=False` | Без изменений |
| **self.ax_phases** | `tick_params(axis='x', labelbottom=True)` — метки под phases | `tick_params(axis='x', labelbottom=False)` — phases **без** подписей X |

**Форматтер времени:** по-прежнему задаётся на `self.ax` в `xaxistosm()` (`self.ax.xaxis.set_major_formatter(ticker.FuncFormatter(self.formtime))`). Оси связаны через `sharex=self.ax`; подписи по X отображаются только у той оси, у которой включён `labelbottom` — теперь только у roast.

### 1.3 Controls ближе к roast (GridSpec)

**Где:** оба вызова `GridSpec(4, 1, ...)` в roast-like ветках.

**Было:** `height_ratios=[10, 0.5, 2.5, 0.7]`, `hspace=0.02`

**Стало:** `height_ratios=[10, 0.5, 2.0, 0.7]`, `hspace=0.01`

- Высота блока controls уменьшена (2.5 → 2.0).
- Вертикальный зазор между subplot’ами уменьшен (0.02 → 0.01).

### 1.4 Комментарии и docstring

- В блоке про roast layout: «X-axis time labels appear only on ax_phases» заменён на: «X-axis time labels appear under roast (ax) only; ax_controls and ax_phases have no x labels».
- В `is_split_layout()`: «ax_phases: phase bar with time labels» заменён на: «ax_phases: phase bar (no x-axis labels)».

### 1.5 Hover / vline / bubble

- Оси по-прежнему создаются с `sharex=self.ax`.
- vline и логика hover не менялись; используется `ax_controls.get_ylim()` — с новым `ylim(-10, 110)` диапазон просто расширен.

### Сводная таблица правок (layout)

| Область | Изменение |
|--------|------------|
| **ax_controls** (оба пути) | `set_ylim(-10, 110)`; `set_yticks([0, 25, 50, 75, 100])`; `yaxis.set_minor_locator(ticker.NullLocator())`; убран minor grid |
| **GridSpec** (оба пути) | `height_ratios=[10, 0.5, 2.0, 0.7]`, `hspace=0.01` |
| **self.ax** | В первом блоке: `tick_params(axis='x', labelbottom=True)`; в forceRenewAxis: `show_roast_time_labels = self.roast_layout_like` и передача в `labelbottom=show_roast_time_labels` |
| **self.ax_controls** | Без изменений по X: уже `labelbottom=False` |
| **self.ax_phases** | `tick_params(axis='x', labelbottom=False)` в обоих путях |
| **Formatter** | Без изменений; остаётся на `self.ax` в `xaxistosm()` |

---

## 2. Hover bubble: канонический порядок, BG-пары, bg-only

### Цель

- Строки bubble в **каноническом** порядке; BG строка сразу после своей активной (Air → BG_Air, Drum → BG_Drum и т.д.).
- **bg-only** режим: при отсутствии активного профиля показывать только BG_ строки (roast + controls) по флагам UI.
- Единый фильтр «включён ли BG для ключа» по UI (background ET/BT, Delta, showEtypes, extra visibility).

### 2.1 Канонический порядок строк — ordered_keys

**Где:** в начале `build_hover_bubble_lines()`.

Введён явный список порядка вывода:

```python
ordered_keys: list[str] = [
    'roast_bt', 'roast_et', 'roast_ror_et', 'roast_ror_bt',
    'extra_0', 'extra_1',
    'control_0', 'control_1', 'control_2', 'control_3',
]
```

Строка «Time» по-прежнему выводится первой (отдельно от `ordered_keys`). Итоговый список строк строится только по этому порядку.

### 2.2 Фильтр «включён ли BG для ключа» — bg_enabled_for_raw_key

**Где:** метод уже существовал (~4569–4590); логика не менялась.

- **roast_et** → `self.backgroundETcurve`
- **roast_bt** → `self.backgroundBTcurve`
- **roast_ror_et** → `self.DeltaETBflag`
- **roast_ror_bt** → `self.DeltaBTBflag`
- **control_i** (i = 0..3) → `self.showEtypes[i]`
- **extra_0** → `xtcurveidx > 0` и `extra_channel_visible(xtcurveidx)`
- **extra_1** → `ytcurveidx > 0` и `extra_channel_visible(ytcurveidx)`

Используется при решении, добавлять ли BG_ строку для данного `raw_key`.

### 2.3 Сбор активных значений в active_map

**Где:** `build_hover_bubble_lines()` после заполнения `_hover_line_map`.

- Логика расчёта активных значений **та же**, что раньше (roast по `ax`/`delta_ax`, controls по `ax_controls`), но результат сохраняется в словарь по **raw_key**:
  - **active_map:** `dict[str, tuple[str, str]]` — `raw_key -> (text, color)` для метрик с валидным значением в `t_sec`.
- **has_active:** `has_active = bool(active_map)` — есть ли хотя бы одна активная метрика (иначе считаем режим bg-only).

### 2.4 Формирование итогового списка строк — expanded

**Где:** `build_hover_bubble_lines()`, после расчёта `has_active` и `include_background`.

Алгоритм:

1. **expanded** инициализируется одной строкой времени: `[('time', time_text, time_color)]`.
2. Для каждого **raw_key** из **ordered_keys**:
   - если **has_active** и **raw_key in active_map** — добавляется активная строка: `(raw_key, text, color)`;
   - затем, если **include_background** и **bg_enabled_for_raw_key(raw_key)**:
     - `bg = _bg_value_at(raw_key, t_sec)`; если `bg is not None` — добавляется строка `(raw_key + '_bg', 'BG_' + suffix, bg_color)`.

В результате:

- В режиме **active + BG**: для каждого ключа в каноническом порядке идёт активная строка (если есть), сразу под ней — BG_ строка (если фон включён и ключ разрешён).
- В режиме **bg-only** (`not has_active`): активные строки не добавляются; в bubble попадают только время и BG_ строки по `ordered_keys` и флагам UI.

После этого выполняется `line_specs = expanded`, и дальше без изменений применяется порядок/фильтр из `hover_bubble_config` и формируется `result`.

### 2.5 _bg_value_at: алиасы raw_key

**Где:** начало метода `_bg_value_at()`.

Добавлена нормализация входящего ключа:

- `raw_key == 'BT'` → обрабатывается как `'roast_bt'`
- `raw_key == 'ET'` → обрабатывается как `'roast_et'`

В docstring указано, что принимаются ключи: `roast_bt`, `roast_et`, `roast_ror_et`, `roast_ror_bt`, `extra_0`, `extra_1`, `control_0`..`control_3`, плюс алиасы BT/ET.

### Сводная таблица правок (hover bubble)

| Элемент | Где | Что сделано |
|--------|-----|--------------|
| **ordered_keys** | Начало `build_hover_bubble_lines()` | Список из 10 ключей в фиксированном порядке (roast → extra → control) |
| **bg_enabled_for_raw_key** | Уже в коде | Используется без изменений; один источник правды для «показывать ли BG» по UI |
| **active_map** | `build_hover_bubble_lines()` | Словарь `raw_key -> (text, color)`; заполняется теми же циклами по осям, что раньше давали `line_specs` |
| **has_active** | Там же | `has_active = bool(active_map)` |
| **expanded** | Там же | Сборка: time + для каждого `ordered_keys` — при наличии active строка, затем при включённом BG и разрешённом ключе — BG_ строка; в bg-only активные не добавляются |
| **line_specs** | Там же | `line_specs = expanded`; далее без изменений — config order/filter и `result` |
| **_bg_value_at** | Начало метода | Алиасы BT→roast_bt, ET→roast_et; обновлён docstring |

---

## Файлы

- Все изменения — в **`src/artisanlib/canvas.py`** (roast-like layout в `redraw_roast_like()` и связанных блоках; hover bubble в `build_hover_bubble_lines()` и `_bg_value_at()`).
- Отчёт сохранён в **`docs/CHANGELOG_CANVAS_LAYOUT_AND_HOVER.md`**.
