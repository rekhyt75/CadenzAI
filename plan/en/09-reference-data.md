# 09 · Reference data, templates and tokens

## 1. `config/profile.yaml` template

```yaml
language: en            # UI and generated text language
units: metric           # metric | imperial
garmin:
  key_file: ~/.config/garmin/connect-personal-api-key
  export_dir: null      # path of the unzipped Garmin data export, or null
athlete:
  weight_kg: null       # null = use the latest Garmin weight value
  birth_year: null      # optional, age context only
  max_hr: null          # null = take it from the device zones (preferred)
history:
  start_date: null      # first date to import; null = first ride with a power meter
defaults:
  period: 6m            # all | 2y | 1y | 6m | 3m | 1m
  pause_threshold_days: 10
  return_days_after_injury: 14
server:
  host: 127.0.0.1
  port: 8765
assumptions:            # shown in the UI; edit to your setup
  system_mass_kg: null  # rider + bike + kit; null = weight + 13 kg
  crr: 0.012
  cda_m2: 0.5
  air_density: 1.2
  torque_ref_cadence_rpm: 80
bikes: []               # optional: [{name, kind, status, bought_on}]
ai_summary:             # OPTIONAL local-AI ride summary (see 10); off unless the user says yes
  enabled: false
  model: qwen2.5:7b
  url: http://127.0.0.1:11434   # must be a loopback address
```

## 2. Power percentile reference (men, W/kg)

Source: a published empirical distribution of power-meter users as reproduced by `https://www.adaptcycling.com/tools/power-profile` (read on 20 Nov 2026; the page does not name the dataset). Coggan's original power-profile table is the usual alternative: if you can read it, prefer it and cite it. **Re-fetch and verify before use.** No 60-minute column and no data below the 30th percentile.

| Tier | Percentile | 5 s | 1 min | 5 min | 20 min |
|---|---|---|---|---|---|
| Trained | 30th | 10.43 | 5.73 | 3.99 | 3.38 |
| Competitive | 55th | 12.64 | 6.71 | 4.61 | 3.90 |
| Strong amateur | 75th | 14.71 | 7.59 | 5.15 | 4.37 |
| Elite amateur | 90th | 16.74 | 8.55 | 5.73 | 4.86 |
| Top amateur | 98th | 19.01 | 9.58 | 6.43 | 5.41 |

## 3. Rider-type shapes (author's hypotheses, **not measurements**)

Relative weight of each axis for a type; the values are normalised to mean 1 and scaled to the user's mean score before drawing, so only the **shape** is compared. Axis order: VO₂max, threshold, sprint, endurance, climbing, flat, recovery. Let the user edit them.

| Type | VO₂max | Threshold | Sprint | Endurance | Climbing | Flat | Recovery |
|---|---|---|---|---|---|---|---|
| Sprinter | 0.85 | 0.65 | 1.90 | 0.70 | 0.55 | 1.00 | 1.00 |
| Puncheur | 1.20 | 1.00 | 1.30 | 0.90 | 1.00 | 0.80 | 1.10 |
| MTB / XC | 1.30 | 1.10 | 1.20 | 1.00 | 1.20 | 0.60 | 1.20 |
| Climber | 1.25 | 1.20 | 0.70 | 1.10 | 1.50 | 0.70 | 1.00 |
| Time-trialist / rouleur | 0.90 | 1.30 | 0.80 | 1.20 | 0.70 | 1.50 | 1.00 |
| Endurance / gran fondo | 0.90 | 1.10 | 0.60 | 1.50 | 1.00 | 1.10 | 1.00 |
| All-rounder | 1 | 1 | 1 | 1 | 1 | 1 | 1 |

(In W/kg a time-trialist resembles a climber; the real distinction is in absolute watts, which the radar does not show.)

## 4. Colour tokens

Define as CSS variables; the dark set must be declared **both** under `@media (prefers-color-scheme: dark)` (when no explicit light theme is chosen) **and** under `:root[data-theme="dark"]`.

| Token | Light | Dark |
|---|---|---|
| page / surface | `#f9f9f7` / `#fcfcfb` | `#0d0d0d` / `#1a1a19` |
| ink / secondary / muted | `#0b0b0b` / `#52514e` / `#6b6a65` | `#f0efec` / `#c3c2b7` / `#a09f98` |
| grid / axis | `#e1e0d9` / `#c3c2b7` | `#2c2c2a` / `#383835` |
| series 1 … 8 (categorical, **fixed order**) | `#2a78d6 #eb6834 #1baf7a #eda100 #e87ba4 #008300 #6250d6 #e34948` | `#3987e5 #d95926 #199e70 #c98500 #d55181 #0ca30c #9085e9 #e66767` |
| power zones Z1…Z7 (sequential) | `#86b6ef #6da7ec #5598e7 #3987e5 #256abf #184f95 #0d366b` | `#184f95 #256abf #2a78d6 #3987e5 #6da7ec #9ec5f4 #cde2fb` |
| HR zones Z1…Z5 (sequential) | `#86b6ef #5598e7 #2a78d6 #1c5cab #0d366b` | `#184f95 #2a78d6 #5598e7 #9ec5f4 #cde2fb` |
| diverging negative / mid / positive | `#e34948 / #f0efec / #2a78d6` | `#e66767 / #383835 / #3987e5` |
| status good / warning / serious / critical | `#006300 / #fab219 / #ec835a / #d03b3b` | `#0ca30c / #fab219 / #ec835a / #e66767` |

Only the **first three** categorical slots are guaranteed distinguishable in every chart type (scatter/bubble); beyond three, fold into "Other" or use small multiples. Validate any palette you swap in (colour-blind simulation, contrast against each surface) before shipping.

## 5. FIT messages and fields used

| Message | Fields |
|---|---|
| `record` | `timestamp, position_lat, position_long, power, heart_rate, cadence, enhanced_speed, enhanced_altitude, distance, temperature, left_right_balance, left/right_torque_effectiveness, left/right_pedal_smoothness` |
| `lap` | `total_distance, total_timer_time, avg_power, normalized_power, max_power, avg_heart_rate, avg_cadence, enhanced_avg_speed, total_ascent, total_descent, avg_temperature, start_time, lap_trigger, total_work` |
| `session` | `total_work` (joules), `avg_cadence, avg_temperature, normalized_power, threshold_power, left_right_balance, avg_left/right_torque_effectiveness, avg_left/right_pedal_smoothness, time_standing, total_calories` |
| `device_info` | `manufacturer, product, serial_number, device_type, antplus_device_type, source_type` |
| `file_id` | `time_created` (UTC), `type` |

## 6. Dependencies (pin exact versions in your lock files)

Backend: `fastapi, uvicorn, httpx, fitparse, numpy, scipy, pydantic`. Frontend: `react, react-dom, recharts (2.x), vite, @vitejs/plugin-react`. Optional for verification: a browser-automation tool (Chrome DevTools MCP or Playwright). Frontend map: `leaflet` (1.9). Optional local AI (`10`): the Ollama program and one model (reference: `qwen2.5:7b`), not Python packages.
