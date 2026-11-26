# 09 · Dati di riferimento, template e token

## 1. Template di `config/profile.yaml`

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
  model: qwen3:4b-instruct
  url: http://127.0.0.1:11434   # must be a loopback address
```

## 2. Riferimento dei percentili di potenza (uomini, W/kg)

Fonte: una distribuzione empirica pubblicata degli utenti di misuratori di potenza, così come riprodotta da `https://www.adaptcycling.com/tools/power-profile` (letta il 20 Nov 2026; la pagina non indica il dataset). La tabella originale del power profile di Coggan è l'alternativa abituale: se riesci a leggerla, preferiscila e citala. **Scarica di nuovo e verifica prima dell'uso.** Nessuna colonna a 60 minuti e nessun dato sotto il trentesimo percentile.

| Livello | Percentile | 5 s | 1 min | 5 min | 20 min |
|---|---|---|---|---|---|
| Allenato | 30th | 10.43 | 5.73 | 3.99 | 3.38 |
| Competitivo | 55th | 12.64 | 6.71 | 4.61 | 3.90 |
| Amatore forte | 75th | 14.71 | 7.59 | 5.15 | 4.37 |
| Amatore élite | 90th | 16.74 | 8.55 | 5.73 | 4.86 |
| Amatore di vertice | 98th | 19.01 | 9.58 | 6.43 | 5.41 |

## 3. Forme dei tipi di ciclista (ipotesi dell'autore, **non misure**)

Peso relativo di ciascun asse per un tipo; i valori sono normalizzati a media 1 e scalati sul punteggio medio dell'utente prima del disegno, quindi si confronta solo la **forma**. Ordine degli assi: VO₂max, soglia, sprint, resistenza, salita, pianura, recupero. Lascia che l'utente li modifichi.

| Tipo | VO₂max | Soglia | Sprint | Resistenza | Salita | Pianura | Recupero |
|---|---|---|---|---|---|---|---|
| Velocista | 0.85 | 0.65 | 1.90 | 0.70 | 0.55 | 1.00 | 1.00 |
| Puncheur | 1.20 | 1.00 | 1.30 | 0.90 | 1.00 | 0.80 | 1.10 |
| MTB / XC | 1.30 | 1.10 | 1.20 | 1.00 | 1.20 | 0.60 | 1.20 |
| Scalatore | 1.25 | 1.20 | 0.70 | 1.10 | 1.50 | 0.70 | 1.00 |
| Cronoman / passista | 0.90 | 1.30 | 0.80 | 1.20 | 0.70 | 1.50 | 1.00 |
| Fondista / granfondo | 0.90 | 1.10 | 0.60 | 1.50 | 1.00 | 1.10 | 1.00 |
| Completo | 1 | 1 | 1 | 1 | 1 | 1 | 1 |

(In W/kg un cronoman somiglia a uno scalatore; la vera distinzione sta nei watt assoluti, che il radar non mostra.)

## 4. Token di colore

Definiscili come variabili CSS; il set scuro va dichiarato **sia** sotto `@media (prefers-color-scheme: dark)` (quando non è scelto un tema chiaro esplicito) **sia** sotto `:root[data-theme="dark"]`.

| Token | Chiaro | Scuro |
|---|---|---|
| pagina / superficie | `#f9f9f7` / `#fcfcfb` | `#0d0d0d` / `#1a1a19` |
| inchiostro / secondario / attenuato | `#0b0b0b` / `#52514e` / `#6b6a65` | `#f0efec` / `#c3c2b7` / `#a09f98` |
| griglia / asse | `#e1e0d9` / `#c3c2b7` | `#2c2c2a` / `#383835` |
| serie 1 … 8 (categoriche, **ordine fisso**) | `#2a78d6 #eb6834 #1baf7a #eda100 #e87ba4 #008300 #6250d6 #e34948` | `#3987e5 #d95926 #199e70 #c98500 #d55181 #0ca30c #9085e9 #e66767` |
| zone di potenza Z1…Z7 (sequenziali) | `#86b6ef #6da7ec #5598e7 #3987e5 #256abf #184f95 #0d366b` | `#184f95 #256abf #2a78d6 #3987e5 #6da7ec #9ec5f4 #cde2fb` |
| zone di FC Z1…Z5 (sequenziali) | `#86b6ef #5598e7 #2a78d6 #1c5cab #0d366b` | `#184f95 #2a78d6 #5598e7 #9ec5f4 #cde2fb` |
| divergente negativo / centro / positivo | `#e34948 / #f0efec / #2a78d6` | `#e66767 / #383835 / #3987e5` |
| stato buono / avviso / grave / critico | `#006300 / #fab219 / #ec835a / #d03b3b` | `#0ca30c / #fab219 / #ec835a / #e66767` |

Solo i **primi tre** slot categorici sono garantiti distinguibili in ogni tipo di grafico (scatter/bubble); oltre i tre, raggruppa in "Altro" o usa piccoli multipli. Convalida qualsiasi palette che sostituisci (simulazione del daltonismo, contrasto rispetto a ciascuna superficie) prima del rilascio.

## 5. Messaggi e campi FIT usati

| Messaggio | Campi |
|---|---|
| `record` | `timestamp, position_lat, position_long, power, heart_rate, cadence, enhanced_speed, enhanced_altitude, distance, temperature, left_right_balance, left/right_torque_effectiveness, left/right_pedal_smoothness` |
| `lap` | `total_distance, total_timer_time, avg_power, normalized_power, max_power, avg_heart_rate, avg_cadence, enhanced_avg_speed, total_ascent, total_descent, avg_temperature, start_time, lap_trigger, total_work` |
| `session` | `total_work` (joules), `avg_cadence, avg_temperature, normalized_power, threshold_power, left_right_balance, avg_left/right_torque_effectiveness, avg_left/right_pedal_smoothness, time_standing, total_calories` |
| `device_info` | `manufacturer, product, serial_number, device_type, antplus_device_type, source_type` |
| `file_id` | `time_created` (UTC), `type` |

## 6. Dipendenze (fissa le versioni esatte nei tuoi lock file)

Backend: `fastapi, uvicorn, httpx, fitparse, numpy, scipy, pydantic`. Frontend: `react, react-dom, recharts (2.x), vite, @vitejs/plugin-react`. Opzionale per la verifica: uno strumento di automazione del browser (Chrome DevTools MCP o Playwright). Mappa nel frontend: `leaflet` (1.9). IA locale facoltativa (`10`): il programma Ollama e un modello (riferimento: `qwen3:4b-instruct`), non pacchetti Python.
