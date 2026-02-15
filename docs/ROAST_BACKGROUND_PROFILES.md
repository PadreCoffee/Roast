# Roast-like layout: фоновые профили (background curves)

## Задача

Сделать так, чтобы background profiles / background curves (фоновое сравнение с другими профилями) рисовались по новой схеме roast-like layout: верхний график `ax` (BT/ET/extra + RoR + маркеры событий), нижний `ax_controls` (step-линии Air/Drum/Burner/Power), снизу `ax_phases`.

## Изменённый файл

- **`src/artisanlib/canvas.py`**

---

## Что сделано

### 1. Контейнеры для фоновых линий (разделение active / background)

Добавлены три списка, чтобы фоновые artists можно было удалять/перерисовывать при redraw без мусора и без дублей:

- **`self._bg_roast_lines`** — линии фона на `ax` (BT, ET, extra devices)
- **`self._bg_controls_lines`** — линии фона на `ax_controls` (E1–E4 step)
- **`self._bg_delta_lines`** — линии фона дельт (RoR ET/BT) на `delta_ax`

В начале `redraw_roast_like()` при очистке осей (`ax.clear()` и при `fig.clf()` при `forceRenewAxis`) эти списки очищаются.

### 2. Роутинг фона на новые оси

| Тип фона | Куда рисуется | Примечание |
|----------|----------------|------------|
| Roast background | **`self.ax`** | BT/ET/extra с теми же осями/масштабом/таймскейлом, что и активный профиль |
| Controls background | **`self.ax_controls`** | E1–E4 step-линии (0–100%), синхронно по X, без подписей/аннотаций |
| Delta (RoR) background | **`self.delta_ax`** (через `transData`) | При включённых DeltaETBflag/DeltaBTBflag |

### 3. Визуальный стиль фона

- **linewidth:** 0.8 (тоньше активных)
- **alpha:** 0.35 (в диапазоне 0.25–0.4)
- **zorder:** 2 (ниже активных и hover)
- **animated:** False
- Без маркеров (`marker='None'`, `markersize=0`), без path_effects

### 4. Исключение фона из hover

- У всех фоновых линий **label** начинается с `'_'**: `'_bg_ET'`, `'_bg_BT'`, `'_bg_XT'`, `'_bg_YT'`, `'_bg_E1'`…`'_bg_E4'`, `'_bg_DeltaET'`, `'_bg_DeltaBT'`.
- В `build_hover_bubble_lines()` уже есть проверка `lbl.startswith('_')` — такие линии не попадают в `_hover_line_map` и в bubble. Дополнительных motion handlers не добавлялось.

### 5. Кэш декодирования фона

- Данные фона (`timeB`, `stemp1B`, `stemp2B`, массивы E1–E4) берутся только из:
  - **loadbackground()** (при загрузке профиля),
  - и при **redraw_roast_like** (при необходимости — пересчёт сглаживания при `re_smooth_background`).
- В обработчике движения мыши (onmove/hover) **никакого декодирования или пересчёта массивов фона не выполняется**.

### 6. Пересборка при redraw и смене layout

- При каждом `redraw_roast_like()` в начале очищаются `_bg_*`, затем при включённом фоне заново строятся только фоновые линии.
- При `ax.clear()` или `fig.clf()` списки `_bg_*` очищаются, чтобы не оставалось «залипших» artists на старых осях.

---

## Где была старая схема и как теперь

- **Раньше:** при `roast_layout_like=True` вызывался только `redraw_roast_like()` и выходил; блок классического redraw с отрисовкой фона (l_back1, l_back2, E1–E4 background на ax/ax_controls) не выполнялся — **в roast-like режиме фона не было**.
- **Теперь:** фон в roast-like рисуется **внутри** `redraw_roast_like()` после `_draw_phases_bar()` и до отрисовки событий и переднего плана: roast-кривые на `ax`, controls на `ax_controls`, дельты через `delta_ax.transData`, с отдельными контейнерами `_bg_roast_lines`, `_bg_controls_lines`, `_bg_delta_lines`.

---

## Проверка (Acceptance)

1. **Включить background curves** — на `ax` видны фоновые BT/ET/extra в фоне; на `ax_controls` видны фоновые step-линии controls.
2. **Hover** — работает как раньше, показывает только активный профиль (фоновые линии не участвуют в bubble).
3. **Открыть/закрыть настройки, переключить roast-like layout** — background перерисовывается корректно, без дублей и без «оторванных» artists.
4. **Отключить background** — фоновые линии полностью исчезают (контейнеры очищаются, блок не рисует).

---

## Ограничения

- Поддерживается один загруженный фоновый профиль (как и в классическом режиме). Несколько фоновых профилей не реализованы (при необходимости можно ограничить N или группировать).
