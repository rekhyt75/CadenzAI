# 02 · Data model (SQLite) and file storage

One SQLite file (`data/<app>.db`, `PRAGMA` defaults are fine) plus the original FIT files. Create tables with `CREATE TABLE IF NOT EXISTS` and **migrate additively** (`ALTER TABLE … ADD COLUMN` when a column is missing): the app must start on a database created by an older version. Never use `DROP`/`DELETE` for age. All dates are ISO strings; ride start times are stored in **local time** (`start_local`) and streams carry their own UTC `t0`.

## 1. Core tables

| Table | Key | Content |
|---|---|---|
| `activities` | `id` (Garmin activity id) | `name, start_local, type_key` (e.g. `road_biking`, `gravel_cycling`, `mountain_biking`, `indoor_cycling`, `virtual_ride`), `distance_m, duration_s, moving_s, elev_gain_m, avg_hr, max_hr, avg_power, max_power, np, tss, intensity_factor, ftp` (FTP stored on that ride by Garmin), `work_kj, sensors` (JSON list of external sensors), `detail_loaded, fit_loaded` flags, bike columns (below) |
| `rec` | `activity_id` | `n`, `blob` = zlib-compressed JSON of the decoded 1 Hz stream (section 3) |
| `laps` | `(activity_id, idx)` | `data` = JSON of the FIT lap message (non-null, non-"unknown" fields) |
| `session` | `activity_id` | JSON of the FIT session message (includes pedal dynamics averages and total work) |
| `devices` | `activity_id` (many rows) | JSON of each FIT `device_info` message (manufacturer, serial, type, source) |
| `best_power` | `(activity_id, seconds)` | best mean power per duration for every ride, from the cleaned stream |
| `power_quality` | `activity_id` | `spikes` (samples discarded), `max_spike_w`, `no_power` |
| `derived` | `activity_id` | seconds in power zones `pz0..pz7` and heart-rate zones `hz0..hz5`, `p_secs`, `h_secs`, `tss_own, tss_diff_pct, hr_max_obs` |
| `climbs` | none (many per ride) | `activity_id, start_t, dur_s, dist_m, gain_m, grade, avg_power, avg_hr, vam, lat, lon` |
| `ride_load` | `activity_id` | `descent_m, descent_h, brake_kj, pedal_kj, hours` (estimated loads for wear indices) |
| `profile_ride` | `activity_id` | `late20_w` (best 20 min after 1500 kJ), `flat20_w` (best 20 min on near-flat ground), `work_kj` |
| `ride_stats` | `activity_id` | `vi, decoup_pct, ef1, ef2, dec_min, cad_med, torque_med, q1..q4, pedal_secs` |
| `ride_quad` | `activity_id` | sparse JSON `{"cadBin,torqueBin": seconds}` for the torque-vs-cadence map |

## 2. Wellness, bikes, pauses

| Table | Key | Content |
|---|---|---|
| `wellness_daily` | `day` | `resting_hr, min_hr, max_hr, steps, moderate_min, vigorous_min, stress_avg, bb_high, bb_low, bb_start, bb_sleepstart, respiration_hi/lo, kcal_active, source` (`export` or `api`) |
| `sleep` | `day` | stage seconds, `score`, respiration, `sleep_stress`, `awake_count`, `feedback` |
| `hrv` | `day` | `last_night_avg, weekly_avg, high5min, status, balanced_low, balanced_upper` |
| `vo2max` | `day` | `value` (precise), `rounded`, `scope` (`ciclismo` or `generico`) |
| `athlete` | `id` (always 1) | `sex`, `birth_year`, `birth_date`, `weight_kg`, `hr_max`, `bike_gear_kg`, `hr_zone_floors` (JSON), `lthr`, `hr_source`, `source`: the user's profile, local only, never in the repository |
| `wellness_extra` | `day` | `stress_rest_s, stress_low_s, stress_med_s, stress_high_s, stress_level, stress_qualifier, resp_avg, resp_max, resp_min, bb_charged, bb_drained, bb_latest` |
| `sleep_extra` | `day` | `overall_q` (Garmin's class), `scores` (JSON with the class and optimal range of each item), `nap_s`, `insight` |
| `fitness_age` | `day` | `chrono, fitness, achievable, vigorous_days, vigorous_min, rhr, bmi` |
| `ref_power_age` | `duration_s, sex, age_band` | `low10, mean, top10, source, url` |
| `bikes` | `id` | `name` (unique), `kind` (`mtb`, `gravel`, `strada`/road, `rulli`/indoor, `altro`), `status` (`attiva`/`in_vendita`/`venduta`), `bought_on, sold_on, price, list_price, sold_price, notes, color_slot` (1–8, **fixed per bike forever**) |
| `components` | `id` | `bike, kind, label, model, source, installed_on, removed_on, price, vendor, threshold_km, threshold_h, threshold_elev, threshold_days, notes, is_upgrade, upgrade_value, original` |
| `pauses` | `start` | `end, reason, note, manual` (only user annotations are stored; automatic pauses are computed) |
| `meta` | `key` | free key/value |

Bike columns on `activities`: `bike` (name), `bike_manual` (1 = user's choice, **never overwrite**), `bike_uncertain` (1 = to confirm), `bike_note` (why), `garmin_bike` (the name Garmin has, historical only).

## 3. Stream format (`rec.blob`)

Decompress and parse JSON. All arrays have the same length (one entry per record, normally 1 Hz); a missing value is `null`.

```json
{
  "t0": "2026-01-01T07:00:00+00:00",
  "t":  [0, 1, 2],
  "lat": [], "lon": [],
  "p": [], "h": [], "c": [], "v": [], "a": [], "d": [], "T": [],
  "b": [], "te_l": [], "te_r": [], "ps_l": [], "ps_r": []
}
```

`t` seconds from `t0` (UTC); `p` power W; `h` heart rate; `c` cadence rpm; `v` speed m/s; `a` altitude m; `d` cumulative distance m; `T` temperature; `b` left/right balance (raw FIT value, see below); `te_*` torque effectiveness; `ps_*` pedal smoothness. GPS stored as degrees (convert FIT semicircles: `deg = v * 180 / 2**31`).

**Do not drop pauses when decoding**: keep real timestamps. Analyses build a regular 1 Hz grid where missing seconds are `NaN`, so a pause is "missing data", not zeros.

**Left/right balance** in the session message is a raw 16-bit value: bit `0x8000` means "the percentage refers to the **right** pedal", bits `0x3FFF` = percentage × 100. Right % = `(v & 0x3FFF)/100` if the flag is set, else `100 - that`. Verify against Garmin's own displayed value on one ride.

## 4. Conventions

- Seconds are the unit of time inside streams; hours only for display.
- "Moving time" (`moving_s`) excludes stops; use it for anything mechanical (suspension, components); use elapsed time only for the calendar/duration display.
- Percentages stored as fractions unless the column name ends in `_pct`.
- Indexes: `activities(start_local)` at least.
- **Idempotency:** every importer/derivation must be safe to re-run and must skip what is already done (use "missing" flags/joins).
- The DB is the source of truth for what the UI shows; the UI never talks to Garmin.
