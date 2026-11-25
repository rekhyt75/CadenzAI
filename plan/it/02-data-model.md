# 02 · Modello dei dati (SQLite) e archiviazione dei file

Un unico file SQLite (`data/<app>.db`, i valori predefiniti di `PRAGMA` vanno bene) più i file FIT originali. Crea le tabelle con `CREATE TABLE IF NOT EXISTS` e **migra in modo additivo** (`ALTER TABLE … ADD COLUMN` quando manca una colonna): l'app deve avviarsi su un database creato da una versione precedente. Non usare mai `DROP`/`DELETE` per età. Tutte le date sono stringhe ISO; gli orari di inizio delle uscite sono salvati in **ora locale** (`start_local`) e i flussi portano il proprio `t0` in UTC.

## 1. Tabelle principali

| Tabella | Chiave | Contenuto |
|---|---|---|
| `activities` | `id` (id attività Garmin) | `name, start_local, type_key` (ad es. `road_biking`, `gravel_cycling`, `mountain_biking`, `indoor_cycling`, `virtual_ride`), `distance_m, duration_s, moving_s, elev_gain_m, avg_hr, max_hr, avg_power, max_power, np, tss, intensity_factor, ftp` (FTP salvato da Garmin su quell'uscita), `work_kj, sensors` (elenco JSON dei sensori esterni), flag `detail_loaded, fit_loaded`, colonne della bici (sotto) |
| `rec` | `activity_id` | `n`, `blob` = JSON compresso con zlib del flusso decodificato a 1 Hz (sezione 3) |
| `laps` | `(activity_id, idx)` | `data` = JSON del messaggio lap del FIT (campi non nulli e non "unknown") |
| `session` | `activity_id` | JSON del messaggio session del FIT (include le medie della dinamica di pedalata e il lavoro totale) |
| `devices` | `activity_id` (molte righe) | JSON di ogni messaggio `device_info` del FIT (produttore, seriale, tipo, sorgente) |
| `best_power` | `(activity_id, seconds)` | migliore potenza media per durata per ogni uscita, dal flusso ripulito |
| `power_quality` | `activity_id` | `spikes` (campioni scartati), `max_spike_w`, `no_power` |
| `derived` | `activity_id` | secondi nelle zone di potenza `pz0..pz7` e nelle zone di frequenza cardiaca `hz0..hz5`, `p_secs`, `h_secs`, `tss_own, tss_diff_pct, hr_max_obs` |
| `climbs` | nessuna (molte per uscita) | `activity_id, start_t, dur_s, dist_m, gain_m, grade, avg_power, avg_hr, vam, lat, lon` |
| `ride_load` | `activity_id` | `descent_m, descent_h, brake_kj, pedal_kj, hours` (carichi stimati per gli indici di usura) |
| `profile_ride` | `activity_id` | `late20_w` (migliori 20 min dopo 1500 kJ), `flat20_w` (migliori 20 min su terreno quasi piano), `work_kj` |
| `ride_stats` | `activity_id` | `vi, decoup_pct, ef1, ef2, dec_min, cad_med, torque_med, q1..q4, pedal_secs` |
| `ride_quad` | `activity_id` | JSON sparso `{"cadBin,torqueBin": seconds}` per la mappa coppia-vs-cadenza |

## 2. Wellness, bici, pause

| Tabella | Chiave | Contenuto |
|---|---|---|
| `wellness_daily` | `day` | `resting_hr, min_hr, max_hr, steps, moderate_min, vigorous_min, stress_avg, bb_high, bb_low, bb_start, bb_sleepstart, respiration_hi/lo, kcal_active, source` (`export` o `api`) |
| `sleep` | `day` | secondi per fase, `score`, respirazione, `sleep_stress`, `awake_count`, `feedback` |
| `hrv` | `day` | `last_night_avg, weekly_avg, high5min, status, balanced_low, balanced_upper` |
| `vo2max` | `day` | `value` (preciso), `rounded`, `scope` (`ciclismo` o `generico`) |
| `athlete` | `id` (sempre 1) | `sex`, `birth_year`, `birth_date`, `weight_kg`, `hr_max`, `bike_gear_kg`, `hr_zone_floors` (JSON), `lthr`, `hr_source`, `source`: il profilo dell'utente, solo in locale, mai nel repository |
| `wellness_extra` | `day` | `stress_rest_s, stress_low_s, stress_med_s, stress_high_s, stress_level, stress_qualifier, resp_avg, resp_max, resp_min, bb_charged, bb_drained, bb_latest` |
| `sleep_extra` | `day` | `overall_q` (classe di Garmin), `scores` (JSON con classe e intervallo ottimale di ogni voce), `nap_s`, `insight` |
| `fitness_age` | `day` | `chrono, fitness, achievable, vigorous_days, vigorous_min, rhr, bmi` |
| `ref_power_age` | `duration_s, sex, age_band` | `low10, mean, top10, source, url` |
| `bikes` | `id` | `name` (univoco), `kind` (`mtb`, `gravel`, `strada`/strada, `rulli`/indoor, `altro`), `status` (`attiva`/`in_vendita`/`venduta`), `bought_on, sold_on, price, list_price, sold_price, notes, color_slot` (1–8, **fisso per ogni bici per sempre**) |
| `components` | `id` | `bike, kind, label, model, source, installed_on, removed_on, price, vendor, threshold_km, threshold_h, threshold_elev, threshold_days, notes, is_upgrade, upgrade_value, original` |
| `pauses` | `start` | `end, reason, note, manual` (si salvano solo le annotazioni dell'utente; le pause automatiche sono calcolate) |
| `meta` | `key` | chiave/valore libero |

Colonne della bici su `activities`: `bike` (nome), `bike_manual` (1 = scelta dell'utente, **da non sovrascrivere mai**), `bike_uncertain` (1 = da confermare), `bike_note` (perché), `garmin_bike` (il nome che ha Garmin, solo storico).

## 3. Formato del flusso (`rec.blob`)

Decomprimi e analizza il JSON. Tutti gli array hanno la stessa lunghezza (una voce per record, normalmente a 1 Hz); un valore mancante è `null`.

```json
{
  "t0": "2026-01-01T07:00:00+00:00",
  "t":  [0, 1, 2],
  "lat": [], "lon": [],
  "p": [], "h": [], "c": [], "v": [], "a": [], "d": [], "T": [],
  "b": [], "te_l": [], "te_r": [], "ps_l": [], "ps_r": []
}
```

`t` secondi da `t0` (UTC); `p` potenza W; `h` frequenza cardiaca; `c` cadenza rpm; `v` velocità m/s; `a` altitudine m; `d` distanza cumulativa m; `T` temperatura; `b` bilanciamento sinistra/destra (valore FIT grezzo, vedi sotto); `te_*` efficacia della coppia; `ps_*` fluidità della pedalata. Il GPS è salvato in gradi (converti i semicerchi FIT: `deg = v * 180 / 2**31`).

**Non scartare le pause durante la decodifica**: conserva i timestamp reali. Le analisi costruiscono una griglia regolare a 1 Hz in cui i secondi mancanti sono `NaN`, così una pausa è "dato mancante", non zeri.

**Bilanciamento sinistra/destra** nel messaggio session è un valore grezzo a 16 bit: il bit `0x8000` significa "la percentuale si riferisce al pedale **destro**", i bit `0x3FFF` = percentuale × 100. % destra = `(v & 0x3FFF)/100` se il flag è attivo, altrimenti `100 - that`. Verifica con il valore mostrato da Garmin stesso su un'uscita.

## 4. Convenzioni

- I secondi sono l'unità di tempo all'interno dei flussi; le ore solo per la visualizzazione.
- Il "tempo in movimento" (`moving_s`) esclude le soste; usalo per tutto ciò che è meccanico (sospensioni, componenti); usa il tempo trascorso solo per la visualizzazione del calendario/della durata.
- Le percentuali sono salvate come frazioni, a meno che il nome della colonna termini con `_pct`.
- Indici: almeno `activities(start_local)`.
- **Idempotenza:** ogni importatore/derivazione deve poter essere rieseguito senza rischi e deve saltare ciò che è già fatto (usa flag/join "mancante").
- Il DB è la fonte di verità per ciò che mostra l'interfaccia; l'interfaccia non parla mai con Garmin.
