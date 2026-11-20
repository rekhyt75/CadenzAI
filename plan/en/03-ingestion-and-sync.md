# 03 · Ingestion, FIT decoding and sync-on-open

## 1. Pipeline order (per ride)

```text
list (summary)  →  detail  →  FIT file  →  decode  →  clean power  →  best power  →  derived (zones, TSS check, climbs)
   →  loads (descent / braking / pedalling work)  →  ride stats (VI, decoupling, torque map)  →  radar metrics
   →  bike assignment  →  commit
```

Each stage is a function with a "process only what is missing" mode, so the same code serves the first import, the incremental sync and repairs.

## 2. First import

1. **List.** Page the activity list (50 per call) from the chosen start date to *tomorrow*. Keep only cycling types: `cycling, road_biking, gravel_cycling, mountain_biking, indoor_cycling, virtual_ride, e_bike_fitness, e_bike_mountain, cyclocross, track_cycling, bmx, recumbent_cycling, handcycling, indoor_handcycling`. Take the type from `activityType.typeKey`. Insert summary columns (`distance, duration, movingDuration, elevationGain, averageHR, maxHR, avgPower, maxPower, normPower, trainingStressScore, intensityFactor`).
2. **Detail** for every ride without `detail_loaded`: store `functionalThresholdPower` (ride FTP), total work, the sensor list (`metadataDTO.sensors`: manufacturer, device type, serial) and whether an electronic-shifting sensor is present (informational).
3. **FIT file:** from the export if present (see `01`), otherwise `download-service` (ZIP → the single `.fit`). Save the original as `data/fit/<activityId>.fit` and never delete it.
4. **Decode** with `fitparse`: collect `record` messages (sorted by timestamp; timestamps are naive UTC, so attach `tzinfo=UTC` **before** any `.timestamp()` call — mixing local and UTC shifted matching by 1–2 hours once), `lap`, `session`, `device_info`. Drop FIT fields whose name starts with `unknown`. Store the stream as in `02`.
5. Run the derivations of `04` and commit **after every ride**.

Expect long jobs: use a progress log, tolerate `429` (section 4) and make re-runs resume.

## 3. Incremental sync on page open (the "Phase 5" behaviour)

When the server starts (and when the user presses "Update"), in a **background thread** guarded by a lock (so two runs never overlap):

1. Compute `since = date(latest stored ride) - 2 days`; list activities from `since` to tomorrow (1–2 calls). Insert only unknown ids.
2. For each new ride: detail → FIT → decode → derivations → bike assignment (`05`).
3. Wellness: take the last stored day minus 2 days (late data settles) and fetch daily summaries, sleep summaries (chunks of ≤ 28 days) and HRV (chunks of ≤ 180 days) up to today. Upsert.
4. Expose the state at `GET /api/sync` → `{stato: "in corso" | "completato" | "errore", rapporto, errore}` and trigger with `POST /api/sync`. The UI polls every ~3 s while running and shows a one-line status plus a manual "Update" button.
5. Produce a **report of what is not available in new rides** and show it in a warning box: rides without power (excluded from power metrics), without heart rate (excluded from EF and HR zones), with a bike that needs confirmation (the API does not expose gear).

**Test (mandatory):** delete one recent ride from every table and delete its FIT file, run the sync, and verify that exactly that ride is downloaded and that the stored values are identical to before (also check that component wear numbers go down when the ride is removed and come back after).

## 4. Rate limits and robustness

- Minimum 1 s between requests; `429` → wait `Retry-After` or 65 s, retry up to 6 times; other non-200 → raise with the path (never the key).
- A `401` triggers one token refresh and one retry.
- Every network job commits per item; an interrupted run resumes.
- Never hold the DB connection across a long network wait without committing.

## 5. Wellness import

**History from the export** (if provided): daily files and sleep files, only from the chosen start date. Be explicit about field choices (verify each against the API for 3 recent days before trusting it):

- resting HR → `currentDayRestingHeartRate` (not `restingHeartRate`);
- stress average → the `TOTAL` entry of `allDayStress.aggregatorList` (`averageStressLevel`);
- Body Battery high/low/start-of-day/sleep-start → `bodyBattery.bodyBatteryStatList` entries `HIGHEST`, `LOWEST`, `STARTOFDAY`, `SLEEPSTART`;
- sleep score → `sleepScores.overallScore` (export) or `sleepScores.overall.value` (API) — the two sources differ in shape; normalise.

**Recent days from the API:** daily summaries (`heartRate.restingValue`, `movement.steps.value`, `stress.avgLevel`, `bodyBattery.maxValue/minValue`, `intensityMinutes`), sleep summaries, HRV. HRV is **only** available through the API.

Missing values stay `NULL` (never estimated). Report how many days lack sleep, HRV or resting HR in the Data quality page.

## 6. Power-meter and heart-rate absence

Handle rides without power (a missing meter, or a ride on another bike) and without heart rate (dead strap battery) as normal: store them, exclude them from the metrics that need the channel, and show **N/D**. List them in Data quality with the count per month. Do not impute.

## 7. Pitfalls found while building the reference app

- `endDate` is mandatory on the activity list; an omitted one silently returns `[]`.
- `activityType` is an object, not a string.
- The FIT timestamps are UTC; local-time comparison gave "0 of N matched".
- Matching export FIT files by time can produce two candidates when two rides start a minute apart: pick the **nearest** and assert no file is used twice.
- A module named `profile.py` shadows the standard library; avoid names like `profile`, `json`, `types`.
- `PATH` of long jobs: run them in the background and **check the log/DB**, not the launcher's exit message.
