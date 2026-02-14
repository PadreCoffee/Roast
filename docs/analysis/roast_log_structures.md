# Roast/Log Structures (Code-Only)

Источник анализа: код репозитория. Никаких домыслов про единицы или смысл полей без явных указаний.

## Структуры данных

### `ProfileData` (TypedDict, total=False)
Файл: `src/artisanlib/atypes.py`  
Символ: `ProfileData`

Обязательные поля: нет (total=False)

Опциональные поля, связанные с roast/log/samples/events/bbp:

- Time series / samples:
  - `timex: list[float]`
  - `temp1: list[float]`
  - `temp2: list[float]`
  - `samplinginterval: float`
  - `timeindex: list[int]`
  - `phases: list[int]`
  - `extradevices: list[int]`
  - `extratimex: list[list[float]]`
  - `extratemp1: list[list[float]]`
  - `extratemp2: list[list[float]]`
  - `extraname1: list[str]`, `extraname2: list[str]`
  - `extramathexpression1/2: list[str]`
  - `extradevicecolor1/2: list[str]`
  - `extraLCDvisibility1/2: list[bool]`
  - `extraCurveVisibility1/2: list[bool]`
  - `extraDelta1/2: list[bool]`
  - `extraFill1/2: list[int]`
  - `extramarkersizes1/2: list[float]`
  - `extramarkers1/2: list[str]`
  - `extralinewidths1/2: list[float]`
  - `extralinestyles1/2: list[str]`
  - `extradrawstyles1/2: list[str]`
  - `extraNoneTempHint1/2: list[bool]`

- Events:
  - `specialevents: list[int]`
  - `specialeventstype: list[int]`
  - `specialeventsvalue: list[float]`
  - `specialeventsStrings: list[str]`
  - `default_etypes: list[bool]`
  - `default_etypes_set: list[int]`
  - `etypes: list[str]`
  - `anno_positions: list[list[float]]`
  - `flag_positions: list[list[float]]`

- Roast/log metadata:
  - `title: str`, `locale: str`, `beans: str`
  - `weight: list[float|str]`, `volume: list[float|str]`
  - `density: list[float|str]`, `density_roasted: list[float|str]`
  - `defects_weight: float`
  - `roastertype: str`, `roastersize: float`, `roasterheating: int`, `machinesetup: str`
  - `operator: str`, `organization: str`
  - `whole_color: float`, `ground_color: float`, `color_system: str`
  - `roastdate: str`, `roastisodate: str`, `roasttime: str`
  - `roastepoch: int`, `roasttzoffset: int`
  - `roastbatchnr: int`, `roastbatchprefix: str`, `roastbatchpos: int`
  - `roastUUID: str`

- Computed:
  - `computed: ComputedProfileInformation`

- BBP:
  - `bbp_begin: str`
  - `bbp_time_added_from_prev: float`
  - `bbp_endroast_epoch_msec: int`
  - `bbp_endevents: list[list[float|None]]`
  - `bbp_dropevents: list[list[float|None]]`
  - `bbp_dropbt: float`
  - `bbp_dropet: float`
  - `bbp_drop_to_end: float`


### `ComputedProfileInformation` (TypedDict, total=False)
Файл: `src/artisanlib/atypes.py`  
Символ: `ComputedProfileInformation`

Обязательные поля: нет (total=False)

Поля (полный список в коде):
- Temps/Times/Indices: `CHARGE_ET`, `CHARGE_BT`, `TP_idx`, `TP_time`, `TP_ET`, `TP_BT`, `MET`, `DRY_time`, `DRY_ET`, `DRY_BT`, `FCs_time`, `FCs_ET`, `FCs_BT`, `FCe_time`, `FCe_ET`, `FCe_BT`, `SCs_time`, `SCs_ET`, `SCs_BT`, `SCe_time`, `SCe_ET`, `SCe_BT`, `DROP_time`, `DROP_ET`, `DROP_BT`, `COOL_time`, `COOL_ET`, `COOL_BT`
- Phases: `totaltime`, `dryphasetime`, `midphasetime`, `finishphasetime`, `coolphasetime`
- RoR: `dry_phase_ror`, `mid_phase_ror`, `finish_phase_ror`, `total_ror`, `fcs_ror`
- Deltas: `dry_phase_delta_temp`, `mid_phase_delta_temp`, `finish_phase_delta_temp`
- Totals/AUC: `total_ts`, `total_ts_ET`, `total_ts_BT`, `AUC`, `AUCbegin`, `AUCbase`, `AUCfromeventflag`, `dry_phase_AUC`, `mid_phase_AUC`, `finish_phase_AUC`
- Yield/weights: `weight_loss`, `roast_defects_loss`, `total_loss`, `volume_gain`, `moisture_loss`, `organic_loss`, `volumein`, `volumeout`, `weightin`, `weightout`, `roast_defects_weight`, `total_yield`
- Densities & ambient: `green_density`, `roasted_density`, `set_density`, `moisture_greens`, `moisture_roasted`, `ambient_humidity`, `ambient_pressure`, `ambient_temperature`
- RoR deltas: `det`, `dbt`
- Energy/CO2: `BTU_preheat`, `CO2_preheat`, `BTU_bbp`, `CO2_bbp`, `BTU_cooling`, `CO2_cooling`, `BTU_LPG`, `BTU_NG`, `BTU_ELEC`, `BTU_batch`, `BTU_batch_per_green_kg`, `BTU_roast`, `BTU_roast_per_green_kg`, `CO2_batch`, `CO2_batch_per_green_kg`, `CO2_roast`, `CO2_roast_per_green_kg`, `KWH_batch_per_green_kg`, `KWH_roast_per_green_kg`
- BBP metrics: `bbp_total_time`, `bbp_bottom_temp`, `bbp_begin_to_bottom_time`, `bbp_bottom_to_charge_time`, `bbp_begin_to_bottom_ror`, `bbp_bottom_to_charge_ror`


### `BbpCache` (TypedDict, total=False)
Файл: `src/artisanlib/atypes.py`  
Символ: `BbpCache`

Обязательные поля: нет (total=False)

Опциональные поля:
- `mode: str`
- `drop_bt: float`
- `drop_et: float`
- `end_roastepoch_msec: int`
- `end_events: list[list[float|None]]`
- `drop_events: list[list[float|None]]`
- `drop_to_end: float`


### Sync record (roast properties sync)
Файл: `src/plus/roast.py`  
Символы: `sync_record_*`, `getSyncRecord`

Всегда отправляются (даже если None/0):
- `roast_id`, `location`, `coffee`, `blend`, `amount`, `end_weight`, `defects_weight`, `s_item_id`

Опциональные (zero-suppressed):
- `density_roasted`, `batch_number`, `batch_pos`, `whole_color`, `ground_color`, `moisture`
- `temperature`, `pressure`, `humidity`, `roastersize`, `roasterheating`
- `BTU_ELEC`, `BTU_LPG`, `BTU_NG`
- `BTU_roast`, `BTU_preheat`, `BTU_bbp`, `BTU_cooling`, `BTU_batch`
- `CO2_roast`, `CO2_preheat`, `CO2_bbp`, `CO2_cooling`, `CO2_batch`

Опциональные (fifty-suppressed):
- `cupping_score`

Опциональные (empty-string suppressed):
- `label`, `batch_prefix`, `color_system`, `machine`, `notes`, `cupping_notes`


### RoastLog import payload
Файл: `src/artisanlib/roastlog.py`  
Символ: `extractProfileRoastLog`

Используемые поля во входном JSON:
- `line_plots[]`: `channel`, `data`, опционально `label`
- `table_events[]`: `label`, `time`


## Примеры payload (из кода)

### Пример профиля (файл с данными)
Файл: `src/test/sanity/data/loring/RST_LOG1.json`  
Содержит полный JSON профиля, включая `timex`, `temp1`, `temp2`, `specialevents*`, `computed`, `bbp_*`.

### Пример sync record (unit-test)
Файл: `src/test/unitary/plus/test_roast.py`  
Пример входного словаря:
```
{
    'roast_id': 'roast123',
    'location': 'store456',
    'coffee': 'coffee789',
    'amount': 1000,
    'batch_number': 42,
    'label': 'Test Roast',
    'extra_field': 'should_be_ignored',
}
```


## Сериализация и упаковка

- HTTP JSON + gzip:
  - JSON формируется в `sendData()` (compact separators, UTF-8)
  - gzip включается, если `len(jsondata) > post_compression_threshold`
  - `Content-Type: application/json; charset=utf-8`
  - `Content-Encoding: gzip` если сжатие
  - `Idempotency-Key` для POST
  - Файлы: `src/plus/connection.py`, `src/plus/config.py`

- Файловая сериализация `repr`/`ast.literal_eval`:
  - Файл: `src/artisanlib/util.py`
  - Функции: `serialize`, `deserialize`


## UNKNOWN / NOT FOUND

- `roast_session` — NOT FOUND
- `curve` — NOT FOUND
- `sample` как отдельная структура — NOT FOUND (есть только массивы `timex/temp*`)
- Единицы `timex`, `samplinginterval` — UNKNOWN (не указаны явно; в RoastLog используется `d[0]/1000`)
- Коды `specialeventstype` (кроме `4` в RoastLog импорте) — UNKNOWN


## Замечания о целостности

- В RoastLog импорте проверяются длины рядов `timex` и `temp1/temp2`: при несовпадении заполняется `-1.0` длиной `timex`.
- Привязка событий к времени в RoastLog: `table_events[].time` переводится в индекс `timex`.
- `getSyncRecord()` фильтрует данные по whitelist `sync_record_attributes`.
