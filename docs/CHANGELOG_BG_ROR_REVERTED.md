# BG RoR: сделанные изменения и откат

Краткое описание правок по поддержке фоновых RoR (BG_BTror/BG_ETror) в roast-like layout и последующего отката.

---

## Цель (исходный план)

- При включённых **BG_BTror** / **BG_ETror** (флаги `DeltaETBflag` / `DeltaBTBflag`) показывать линии BG RoR на roast-графике (правая ось C/min).
- В hover bubble: при profile+background — сначала Δ профиля (ΔBT/ΔET), под ними BG_ΔBT/BG_ΔET; при background-only — только BG_ΔBT/BG_ΔET при включённых флагах.
- Не менять оси, тики, padding, time-scale — только отрисовка BG RoR и содержимое bubble.

---

## Что было сделано (до отката)

Все правки только в `src/artisanlib/canvas.py`.

### A) Отрисовка BG RoR на roast-графике

1. **Roast-like путь** (около строки 11852):
   - Добавлен комментарий, что здесь рисуются BG RoR линии (правая ось C/min), кеш для bubble — ключи `roast_ror_et` / `roast_ror_bt`.
   - После `ax.plot(...)` для delta1B/delta2B кеш `_bg_hover_cache` заполнялся из **реально отрисованных** данных:
     - `_bg_hover_cache['roast_ror_et'] = (numpy.asarray(ln_d1.get_xdata()).ravel(), numpy.asarray(ln_d1.get_ydata()).ravel())`
     - то же для `roast_ror_bt` и `ln_d2`.
   - Ранее кеш заполнялся так: `(self.timeB, numpy.array(self.delta1B))` и `(self.timeB, numpy.array(self.delta2B))`.

2. **Classic путь** (около строк 13029–13062):
   - После отрисовки `l_delta1B` и `l_delta2B` кеш тоже переводился на данные из линий:
     - `_bg_hover_cache['roast_ror_et']` и `_bg_hover_cache['roast_ror_bt']` из `l_delta1B.get_xdata()/get_ydata()` и `l_delta2B.get_xdata()/get_ydata()`.
   - Добавлены комментарии про «BG RoR hover: cache exactly what was plotted».

### B) Hover bubble

- В **build_hover_bubble_lines**: обновлён комментарий к `ordered_keys` — явно указано, что в bubble есть строки профиля (`roast_ror_et`, `roast_ror_bt`) и BG (`roast_*_bg`).
- В **\_hover_bubble_raw_key_to_config_key**: в докстринге добавлены примеры `roast_ror_et_bg`, `roast_ror_bt_bg` (чтобы было ясно, что ключи `*_bg` не отфильтровываются).

### C) _bg_value_at

- Для веток **roast_ror_et** и **roast_ror_bt** добавлены комментарии: «BG RoR: prefer _bg_hover_cache (roast_ror_et/roast_ror_bt) from plotted line data, then fallback». Логика (сначала кеш) не менялась.

---

## Откат

По запросу пользователя («поломалось все что не надо») все перечисленные изменения откачены.

### Что откатили

1. **Roast-like блок**  
   - Убран комментарий про BG RoR и кеш.  
   - Восстановлено заполнение кеша:  
     `_bg_hover_cache['roast_ror_et'] = (self.timeB, numpy.array(self.delta1B))`  
     `_bg_hover_cache['roast_ror_bt'] = (self.timeB, numpy.array(self.delta2B))`

2. **Classic блок (l_delta1B / l_delta2B)**  
   - Убраны комментарии и возвращено прежнее заполнение кеша из `(self.timeB, numpy.array(self.delta1B))` и `(self.timeB, numpy.array(self.delta2B))`.

3. **build_hover_bubble_lines**  
   - Комментарий к `ordered_keys` возвращён к исходному: «Canonical order of bubble rows (roast, then extra, then controls)».

4. **\_hover_bubble_raw_key_to_config_key**  
   - Докстринг возвращён к: «BG keys (roast_bt_bg etc.) map to same as active.»

5. **\_bg_value_at**  
   - Удалены добавленные комментарии перед проверкой `_bg_hover_cache` для `roast_ror_et` и `roast_ror_bt`.

Итог: поведение и текст в `canvas.py` в этих местах соответствуют состоянию до внедрения описанных правок.

---

## Примечание

Причина поломки после перехода на `get_xdata()/get_ydata()` в кеше не выяснялась в этой сессии. При повторной реализации поддержки BG RoR имеет смысл проверить типы/форматы массивов из `get_xdata()/get_ydata()` (в т.ч. с `numpy.asarray(...).ravel()`) и их использование в `bg_interp_linear` и в отображении bubble.
