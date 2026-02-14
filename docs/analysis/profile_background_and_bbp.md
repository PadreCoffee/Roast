# Profile Background (Overlay) and BBP Findings

This document records confirmed code references for:
1) Profile background/overlay (underlay/reference curve)
2) BBP data in relation to profiles and background

All references include `file + symbol`. No assumptions are made.

## Profile Background / Overlay

### UI entry points (enable/select)

- **Menu item**: `src/artisanlib/main.py::ApplicationWindow.create_roast_menu` adds Background to Roast menu.
- **Action wiring**: `src/artisanlib/main.py::ApplicationWindow.__init__` defines `self.backgroundAction` and connects it to `self.background`.
- **Dialog**: `src/artisanlib/main.py::ApplicationWindow.background` opens `src/artisanlib/background.py::backgroundDlg`.
- **Dialog controls**: `src/artisanlib/background.py::backgroundDlg.readChecks` toggles background visibility and series flags.

### Data source (background)

- **Local file**: `.alog` read via `src/artisanlib/main.py::ApplicationWindow.loadbackground`.
- **Local UUID cache**: `src/artisanlib/main.py::ApplicationWindow.loadbackgroundUUID` resolves UUID to a local path via `src/plus/register.py::getPath` (local shelve cache).

**NO NETWORK FOUND** for background loading.

### Data format (fields)

Data is `ProfileData`:
- `timex`, `temp1`, `temp2`
- `extratimex`, `extratemp1`, `extratemp2`
- `specialevents`, `specialeventstype`, `specialeventsvalue`

Reference: `src/artisanlib/atypes.py::ProfileData`.

### Drawing (series)

Rendering occurs in `src/artisanlib/canvas.py::tgraphcanvas.redraw`:
- Background ET: `self.l_back1` (flag `self.backgroundETcurve`)
- Background BT: `self.l_back2` (flag `self.backgroundBTcurve`)
- Background Delta ET: `self.l_delta1B` (flag `self.DeltaETBflag`)
- Background Delta BT: `self.l_delta2B` (flag `self.DeltaBTBflag`)
- Extra background curves (XT/YT): `self.l_back3`, `self.l_back4` (flags `self.xtcurveidx`, `self.ytcurveidx`)
- Background events annotations: `self.backgroundeventsflag`

Reference: `src/artisanlib/canvas.py::tgraphcanvas.redraw`.

---

## BBP (relation to profile vs. background)

### BBP is part of profile data

Fields stored in `ProfileData`:
- `bbp_begin`, `bbp_time_added_from_prev`
- `bbp_endroast_epoch_msec`
- `bbp_endevents`, `bbp_dropevents`
- `bbp_dropbt`, `bbp_dropet`, `bbp_drop_to_end`

Reference: `src/artisanlib/atypes.py::ProfileData`.

### Loading/saving BBP with a profile

- Load: `src/artisanlib/main.py::ApplicationWindow.loadBbpFromProfile`
- Called during profile load: `src/artisanlib/main.py::ApplicationWindow.setProfile`
- Save into profile: `src/artisanlib/main.py::ApplicationWindow.getProfile`

### Computing BBP metrics

Computed from current profile data in:
`src/artisanlib/main.py::ApplicationWindow.calcBBPMetrics`

### Background vs BBP

There is **no** BBP logic in background dialog or background drawing paths.
- Background dialog: `src/artisanlib/background.py::backgroundDlg` (no BBP fields)
- Background drawing: `src/artisanlib/canvas.py::tgraphcanvas.redraw` (no BBP fields)

Conclusion: BBP is tied to the profile (save/load/compute) and **not** to the profile background overlay.
