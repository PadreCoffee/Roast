# Roast Item Payload (Full Field List)

This document enumerates all fields that can appear in the `roast_item` JSON payload.
Sources are listed per field group and per field where practical.

## Entry Points and Queue

- Entry points (enqueue): `src/plus/controller.py::toggle()`, `src/plus/controller.py::updateSyncRecordHashAndSync()`, `src/artisanlib/roast_properties.py::accept()`, `src/artisanlib/canvas.py::eventaction()`, `src/artisanlib/canvas.py::markDrop()`.
- Queue and send: `src/plus/queue.py::queue_roast_item()`, `src/plus/queue.py::Worker.task()`, `src/plus/connection.py::sendData()`.

## Field Sources (Full List)

### Base template fields from profile data
Source: `src/plus/roast.py::getTemplate()`

Batch/identity/time:
- `batch_number` (from `roastbatchnr`)
- `batch_prefix` (from `roastbatchprefix`)
- `batch_pos` (from `roastbatchpos`)
- `date` (from `roastepoch`, ISO-8601)
- `GMT_offset`
- `id` (from `config.uuid_tag`) -> renamed to `roast_id` in `getRoast()`
- `s_item_id` (from `config.schedule_uuid_tag`)
- `s_item_date` (from `config.schedule_date_tag`)

Weights/density:
- `start_weight`
- `end_weight`
- `defects_weight`
- `density_roasted`

Roaster/ambient/metadata:
- `moisture` (from `moisture_roasted`)
- `label` (from `title`)
- `machine` (from `roastertype`)
- `setup` (from `machinesetup`)
- `roastersize`
- `roasterheating`
- `whole_color`
- `ground_color`
- `color_system`
- `temperature` (from `ambientTemp`)
- `pressure` (from `ambient_pressure`)
- `humidity` (from `ambient_humidity`)

Computed temps/times (profile):
- `charge_temp_ET` (from `CHARGE_ET`)
- `charge_temp` (from `CHARGE_BT`)
- `TP_temp` (from `TP_BT`)
- `DRY_temp` (from `DRY_BT`)
- `FCs_temp` (from `FCs_BT`)
- `FCe_temp` (from `FCe_BT`)
- `drop_temp` (from `DROP_BT`)
- `drop_temp_ET` (from `DROP_ET`)
- `TP_time`
- `DRY_time`
- `FCs_time`
- `FCe_time`
- `drop_time` (from `DROP_time`)
- `FCs_RoR` (from `fcs_ror`)
- `DEV_time` (from `finishphasetime`)
- `DEV_ratio` (from `finishphasetime / totaltime`)
- `AUC`
- `AUC_base` (from `AUCbase`)

### Additional fields added in `getRoast()`
Source: `src/plus/roast.py::getRoast()`

Renames/normalization:
- `roast_id` (from template `id`)
- `amount` (from template `start_weight`)

Computed deltas and energy/CO2:
- `CM_ETD` (from `computed.det`)
- `CM_BTD` (from `computed.dbt`)
- `BTU_ELEC`
- `BTU_LPG`
- `BTU_NG`
- `BTU_roast`
- `BTU_preheat`
- `BTU_bbp`
- `BTU_cooling`
- `BTU_batch`
- `CO2_roast`
- `CO2_preheat`
- `CO2_bbp`
- `CO2_cooling`
- `CO2_batch`

Plus inventory/notes:
- `location` (from `aw.qmc.plus_store`)
- `coffee` (from `aw.qmc.plus_coffee`)
- `blend` (from `aw.qmc.plus_blend_spec` via `trimBlendSpec()`)
- `notes` (from `roastingnotes`)
- `cupping_notes` (from `cuppingnotes`)
- `cupping_score` (from computed flavors)

Background profile (BBP):
- `template` (result of `getTemplate(aw.qmc.backgroundprofile, background=True)`)

File-mod time:
- `modified_at` (from `aw.curFile` modification time)

### Sync record fields (subset used on updates)
Source: `src/plus/roast.py::getSyncRecord()` and `src/plus/roast.py::sync_record_attributes`

Always included:
- `roast_id`
- `location`
- `coffee`
- `blend`
- `amount`
- `end_weight`
- `defects_weight`
- `s_item_id`

Zero-suppressed (synced):
- `density_roasted`
- `batch_number`
- `batch_pos`
- `whole_color`
- `ground_color`
- `moisture`

Zero-suppressed (unsynced, one-way):
- `temperature`
- `pressure`
- `humidity`
- `roastersize`
- `roasterheating`
- `BTU_ELEC`
- `BTU_LPG`
- `BTU_NG`
- `BTU_roast`
- `BTU_preheat`
- `BTU_bbp`
- `BTU_cooling`
- `BTU_batch`
- `CO2_roast`
- `CO2_preheat`
- `CO2_bbp`
- `CO2_cooling`
- `CO2_batch`

Fifty-suppressed:
- `cupping_score`

Empty-string suppressed:
- `label`
- `batch_prefix`
- `color_system`
- `machine`
- `notes`
- `cupping_notes`

## Notes on .alog

- `.alog` is loaded from disk via `src/artisanlib/main.py::loadFile()` but is not attached to payloads.
- Payloads are JSON only: `src/plus/connection.py::sendData()`.
