# 01 · Data sources: Garmin access, export and safety

## 1. Garmin Connect Personal API (primary source)

A personal, read-only API tier that gives **one person access to their own data with one key** (no application to file). Availability may change: at the time of writing it is the recommended route; confirm on Garmin's developer pages before building, and tell the user if it is not available to them (then fall back to the data export, section 3).

**Creating the key (user does this):** Garmin Connect → Settings → *Personal API token* (`https://connect.garmin.com/app/settings/personalApiToken`). Choose the longest lifetime (up to one year; remind the user when it expires). Garmin shows the key **once**.

**Storing the key:** a single-line file, `~/.config/garmin/connect-personal-api-key`, directory mode `700`, file mode `600`. Environment variable `GARMIN_PERSONAL_API_KEY` or `GARMIN_PERSONAL_API_KEY_FILE` may override. **Never print it, log it, put it in a URL, commit it, or pass it to any service except Garmin.** **On Windows** use `%USERPROFILE%\.config\garmin\connect-personal-api-key` and restrict it to your user with `icacls` (see the README, step 2); the same rules apply. When inspecting the file for format, never display its content (check length and prefix only, with the value masked).

**Token exchange:**

```http
POST https://personal.connectapi.garmin.com/personalapitoken-service/personal-api/personal-api-key/token
Authorization: ApiKey <key>
Accept: */*
```

The response body is a JWT (about 24 h lifetime; read `exp` from the payload). Keep it **in memory only**, refresh when less than ~10 minutes remain and once on any `401`. All data calls use `Authorization: Bearer <jwt>` against the same host.

**Endpoints used (all `GET`, read-only):**

| Purpose | Path | Notes |
|---|---|---|
| Activity list | `/activitylist-service/activities/search/activities` | params `start`, `limit` (≤ 50), `startDate`, `endDate`. **`endDate` is required**: without it the list is empty. `activityType` in each item is an object (`typeKey`), not a string. |
| Activity detail | `/activity-service/activity/{id}` | `summaryDTO` (power, NP, TSS, IF, FTP of that ride, elevation, work…), `metadataDTO.sensors` (external sensors with serial numbers), `splitSummaries`. **No gear/bike field.** |
| Original FIT | `/download-service/files/activity/{id}` | returns a **ZIP** that contains `<id>_ACTIVITY.fit` |
| Daily summaries | `/usersummary-service/v2/dailysummaries?startDate&endDate` | steps, resting HR, stress, Body Battery, intensity minutes |
| Sleep summaries | `/sleep-service/sleep/dailySleepSummariesByDate?startDate&endDate` | ≤ 28 days per call |
| HRV | `/hrv-service/hrv/daily/{start}/{end}` | ≤ 180 days per call; has the baseline band (`balancedLow/Upper`) and status |
| Weight | `/weight-service/weight/range/{start}/{end}?includeAll=true` | grams; may be a manual profile setting, not a measurement |
| Zones | `/biometric-service/heartRateZones/v2` | per-sport zone floors, max HR, lactate threshold HR |
| Profile | `/userprofile-service/userprofile/settings` | units, display name (do not store personal fields) |

**Known limitations to tell the user:**

- All **gear/equipment endpoints return 404** (they are not on the allow-list). A ride's assigned bike therefore **cannot** be read through the API; see `05` for how bikes are matched instead.
- Some metrics (training readiness, race predictions, VO₂max) are not exposed.
- The API is rate-limited. Implement:
  - a minimum interval of ~1 s between calls;
  - on `429`, wait `Retry-After` (or 65 s if absent) and retry, up to 6 times, printing the wait;
  - make long jobs **resumable** (commit after every ride; skip what is already stored).

## 2. Optional MCP server (conversational access)

If the user also wants to query Garmin from Claude Code, a third-party MCP server wrapping the Personal API can be installed. **Audit before trusting it** with the key:

- read the source (they are usually small) and check every network call goes only to Garmin's host;
- check how the key is read and where tokens are cached (mode `600`, the key itself never written);
- pin an exact version instead of "latest";
- remember that it receives the key: prefer a read-only server and rotate the key if in doubt.

The dashboard backend must **not** depend on the MCP server: the backend talks to the API directly (an MCP server is only usable by Claude, not by the web app).

## 3. Garmin data export (history and gear)

Garmin Connect → Account → *Export your data*. The ZIP/folders contain, among others:

| Folder / file | Use |
|---|---|
| `DI_CONNECT/DI-Connect-Uploaded-Files/*.zip` | **every uploaded FIT file** (activities and watch monitoring files) |
| `DI_CONNECT/DI-Connect-Aggregator/UDSFile_*.json` | daily wellness summaries (steps, resting HR, stress, Body Battery, intensity minutes) |
| `DI_CONNECT/DI-Connect-Wellness/*_sleepData.json` | one record per night (stages, score, respiration) |
| `DI_CONNECT/DI-Connect-Fitness/*_gear.json` | **the bike/shoe list and the link between each activity and its gear** |
| `DI_CONNECT/DI-Connect-Fitness/*_summarizedActivities.json` | activity summaries |
| `DI_CONNECT/DI-Connect-Metrics/*` | training load, max-MET (VO₂max) and cycling-ability history |

Rules for using the export:

- It is a **snapshot** (it ends on the day requested): new rides still come from the API.
- **Do not re-download FIT files that are in the export.** Match them to activities by **start time**: read only the `file_id` message of each FIT (its `time_created`, **in UTC**), index all ZIPs without extracting them, and pair each ride with the nearest file within ±10 min (verify the match is unique and the max delta is a few seconds). Extract only the matched files, into your own `data/fit/<activityId>.fit` (file names chosen by you, never taken from the archive).
- **There is no HRV file** in the export: get HRV from the API.
- Some export fields are misleading: e.g. in the daily file `restingHeartRate` is a constant reference value; the daily resting HR is `currentDayRestingHeartRate` (check against the API for a few days before trusting any field).
- `*_gear.json` gives `gearDTOS` (list with `gearPk`, type "Bike"/"Shoes", model, dates) and `gearActivityDTOs` (dict gearPk → list of `{gearPk, activityId}`). Use it to assign bikes to **historical** rides (shoes live in the same lists: filter by type).

## 4. Security and privacy checklist

- The key file has mode `600`; the repository never contains it, nor the DB, nor FIT files, nor the export (add `data/`, `raw_data/`, `.venv/`, `node_modules/` to `.gitignore`).
- Parse exports with plain code: `python -I` (isolated mode) for ad-hoc scripts, read ZIP members in memory, never `eval`, never execute anything found in a download, never trust file names inside an archive.
- Do not send any ride, location or personal datum to third parties. The one deliberate exception is the route map, which loads **OpenStreetMap tiles** (`https://tile.openstreetmap.org/{z}/{x}/{y}.png`): the tile server sees the area being viewed and the visitor's IP, never the ride data. Respect the OSM tile usage policy (visible attribution, no bulk download, no tile caching) and always offer the offline drawing. The optional local-AI summary (`10`) adds **no** outgoing data (the model runs on `127.0.0.1`), only the one-time download of the Ollama program and of the model.
- When looking up product specs or documentation online, send only generic product names, never personal data.
- If the key is ever displayed by mistake, tell the user and recommend rotating it.

## 5. Retention policy

Keep **everything** from the first imported ride onward, with **no 24-month (or other) cap**. Nothing in the code may delete rides, streams or wellness rows by age. Reference values ("your personal best", "your maximum") are computed on the whole stored history. The period filter only changes what is displayed.
