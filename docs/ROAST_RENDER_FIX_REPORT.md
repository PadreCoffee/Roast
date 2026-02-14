# Roast-like rendering fix (post-Roboto revert)

**Goal:** Restore stable roast-like rendering to the last "good" state before Roboto changes. No re-introduction of Roboto until rendering is stable.

---

## 1. Debug leaks removed

- **AX_* labels and tinted facecolors**  
  All code that draws big axis labels (`AX_ROAST`, `AX_CONTROLS`, `AX_EVENTS`, `AX_PHASES`) and sets temporary debug facecolors (red/green/blue/yellow tints) is now gated **strictly** by `os.environ.get('ROAST_DEBUG') == '1'`.  
  In normal runs (ROAST_DEBUG unset), **no** debug texts and **no** debug facecolor tints are applied.

- **Logging**  
  - `_cursor_debug_log` (writes to `.cursor/debug.log`) runs only when `ROAST_DEBUG == '1'`.  
  - Per-redraw debug line (axis bounds, BT/ET presence, blit usage) is written to both `~/Library/Logs/Roast/debugging.log` and `./debugging.log` only when `ROAST_DEBUG == '1'`.  
  - ET/BT invariant log and snapshot code remain gated by `_ROAST_DEBUG_ENABLED` (so they still require debug mode).

---

## 2. rcParams / style changes reverted

- **main.py `setFonts()`**  
  All use of `ui_family` / `ui_font_family` in matplotlib `font.family` lists for `graphfont == 0` was removed.  
  Font lists are restored to the pre-Roboto defaults (e.g. Darwin: `['Arial Unicode MS', 'DejaVu Sans', 'sans-serif']` and locale-specific variants) with **no** global UI font prepended.  
  This avoids any rcParams-driven changes to plot rendering (e.g. from a missing or misconfigured Roboto font).  
  **Not changed:** `axes.unicode_minus`, `font.size`, or any theme/facecolor/prop_cycle settings that were not part of the Roboto patch.

---

## 3. Missing ET in Roast-like – root cause and fix

- **Where ET is drawn**  
  `drawET()` uses `self.ax` (same main axis as Roast-like). So ET is drawn on the correct axis.

- **Fixes applied**  
  1. **Color fallback**  
     In `drawET()` (and `drawBT()`), the line color is now `self.palette.get('et') or '#1f77b4'` (ET) and `self.palette.get('bt') or '#d62728'` (BT). If the palette entry is missing or falsy, a visible default is used so the curve is never invisible due to transparent/background color.  
  2. **ROAST_DEBUG invariant**  
     When `ROAST_DEBUG == '1'`, after drawET/drawBT a single JSON line is written (to both debug logs) with: ET/BT presence, visibility, color, alpha, zorder, axes_id, and `axes_is_roast`. This allows quick checks that ET is on the roast axis and visible.  
  3. **Blit disabled for roast-like**  
     With blit disabled for roast-like (see below), the full figure is redrawn each time, avoiding half-updated states where only some axes (or only BT) were updated and ET appeared missing.

- **Root cause (inferred)**  
  Likely a combination of: (a) blit path updating only part of the figure so ET was not consistently repainted, and/or (b) palette or style yielding an ET color that was effectively invisible. The fallback color and disabling blit for roast-like address both.

---

## 4. Blit disabled for roast-like

- In `doUpdate()`, the blit path (`copy_from_bbox` → `update_additional_artists` → `blit`) is **skipped** when `roast_layout_like` is True.  
- For roast-like, the only repaint is the initial `self.fig.canvas.draw()`; no blit is performed.  
- Classic mode is unchanged: blit is still used when not roast-like and when `copy_from_bbox` is available.  
- This makes the roast-like render path deterministic and avoids “half updated” artefacts.

---

## 5. ROAST_DEBUG-only logging

- **Paths**  
  - System: `~/Library/Logs/Roast/debugging.log` (macOS) or `~/.roast/logs/debugging.log` (Linux).  
  - Repo: `./debugging.log` (repo root).  
- **Per-redraw line (when ROAST_DEBUG=1)**  
  One JSON object per redraw, written in `doUpdate()`, containing:  
  `roast_layout_like`, `event_style`, `ax_roast_bounds`, `ax_controls_bounds`, `ax_phases_bounds`, `BT_present`, `ET_present`, `blit_used`.  
- All other debug logs (cursor log, checkpoints, snapshot) run only when debug mode is on (`ROAST_DEBUG == '1'` or `_ROAST_DEBUG_ENABLED`).

---

## 6. Manual checklist

- [ ] **Roast-like ON:** BT and ET curves both visible; no AX_* labels; no tinted debug backgrounds.  
- [ ] **Toggle Roast-like ON/OFF** multiple times; layout and curves remain correct.  
- [ ] **Open 3 profiles** in sequence; curves and layout stay stable.  
- [ ] **Close and reopen app;** persistence does not break rendering (BT/ET and Roast-like layout intact).

---

## Files touched

- `src/artisanlib/canvas.py`: AX_* gate, blit skip for roast-like, ET/BT color fallback, ET/BT invariant log, per-redraw log, `_cursor_debug_log` gating.  
- `src/artisanlib/main.py`: `setFonts()` – removed `ui_family` from all `font.family` lists for graphfont 0.  
- `docs/ROAST_RENDER_FIX_REPORT.md`: this report.
