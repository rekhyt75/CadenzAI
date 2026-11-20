# 00 · Overview and how to run this plan

> **Audience:** Claude Code (and the human driving it). This plan lets you build, from scratch, a **local, single-user cycling analytics dashboard** fed by your own Garmin data. Nothing in this repository is the app itself: it is the specification. Read it fully, ask the questions in section 4, then build the app phase by phase.

## 1. What you are building

A local web app (backend + browser dashboard, nothing leaves the user's computer) that:

- pulls the user's own cycling activities, original FIT files and watch wellness data (sleep, HRV, resting HR, Body Battery) from **Garmin**, automatically and incrementally every time the page is opened;
- stores everything in a local SQLite database plus the original FIT files, with **no age cap** (it keeps growing);
- shows statistics in four areas: **General statistics**, **Calendar**, **Single ride**, **Workshop** (bikes and components);
- tracks **bikes and component wear**, with a second, route-aware wear index (brakes wear on descents, chains on pedalling work…);
- is honest about data quality: it never invents a missing value and always labels estimates as estimates.

Reference feature list (details in the other files):

| Area | Pages |
|---|---|
| General statistics | Physiology (KPIs, volume vs load, peaks, EF, zones, last rides) · Rider profile (7-axis radar + comparison with rider types) · Cardiovascular efficiency (EF, aerobic decoupling) · Power analytics (power-duration curve, FTP/CP estimates) · Training load (CTL/ATL/TSB) · Cadence and torque (quadrants) · Climbs · Evolution (period comparison with a statistical test) · Correlations (load vs watch data) · Recovery (HRV, resting HR, sleep, Body Battery) · Coach (rule-based readings) · Data quality |
| Calendar | Month calendar with rides, week totals and pauses · Rides to match (rides whose bike needs confirmation) · Pauses and reasons |
| Single ride | Ride detail (synchronised charts, route, laps, zones, best efforts, pedal dynamics, quadrants, decoupling) · Browse activities (searchable, sortable table) |
| Workshop | Bike fleet · Components and wear |

## 2. How Claude Code must use this plan

1. **Read** every file in this folder (`00` … `09`) before writing code. The numbering is the reading order, not necessarily the build order (see section 6).
2. **Ask first.** Before the first line of code, check the plan for contradictions or gaps (the original author's rule: *"first verify the whole plan and identify any inconsistencies or points to clarify"*) and ask the human the questions in section 4. Ask only what is indispensable; use sensible defaults for the rest and say so.
3. **Build in phases** (section 6). After each phase run its acceptance checks, using **real data**, and report the outcome truthfully (including failures and anything you skipped).
4. **Never commit or print secrets.** The Garmin API key lives in a file under the user's home directory with mode `600` and is never echoed, logged, committed or sent anywhere except to Garmin.
5. **Treat downloaded files as untrusted** (exports, ZIPs, FIT files, web pages): parse them with safe code, never execute them, extract only into dedicated empty folders.
6. **Respect external services.** Do not bypass bot protections or captchas. If a site blocks automated access, say so and use another public source or ask the user to paste the information.
7. Keep this plan in sync: when a decision changes during the build, record it in the project notes (see `07-operations-and-quality.md`, "Project memory").

## 3. Working rules (analysis and honesty)

Act as a team made of a **cycling performance scientist**, a **professional coach** and a **sports data scientist**. These rules apply to every page, chart and text produced:

- Work only on **cycling** (road, gravel, MTB, indoor). Exclude every other sport.
- **Never invent a missing value.** If a ride has no power or no heart rate, exclude it from the corresponding averages without altering results, and show **N/D** (not available) in tables.
- Write user-facing text in the **user's language** (the reference implementation is Italian; ask the user, and centralise strings in one dictionary so they can be translated).
- **Under every chart, one plain sentence** explaining what it shows.
- Always **separate observation, hypothesis and recommendation.**
- **Do not confuse correlation with causation.** Say it where it matters.
- **Label estimates** (FTP, critical power, climbs, wear indices, anything modelled) as estimates, and state their assumptions.
- If something is **not achievable with the available data**, build the rest and state the limit.
- If you miss a piece of **indispensable** information, ask for it before starting.
- Prefer **measured data over models**, Garmin's own value as the main figure, and your recalculation only as a check (see TSS in `04`).

Engineering rules:

- Keep personal parameters (weight, birth date, heart-rate max, bikes, thresholds) in a profile/config the user can edit — never hard-code them.
- Everything computed from raw streams must be **stored** (derived tables) so pages stay fast, and must be **recomputed incrementally** for new rides.
- Every claim of the app that rests on an assumption must be visible to the user (a "?" help modal, a caption or a note).
- Make destructive or outward-facing actions explicit and reversible; ask before them.

## 4. Questions to ask the user first (customisation)

Collect these in one short round (use the defaults in the last column if the user has no preference). Save the answers in `config/profile.yaml` (template in `09-reference-data.md`).

| # | Question | Default |
|---|---|---|
| 1 | UI/text language (and units: km/m/°C, or miles/ft/°F) | the user's language, metric |
| 2 | How do you access Garmin data? (Personal API key — recommended; optionally the Garmin data export ZIP for history) | Personal API key + one-off export |
| 3 | Body weight (kg) — from Garmin's weight data, or a fixed value | latest Garmin value |
| 4 | Date of birth | used only for age context, optional |
| 5 | Maximum heart rate to use for zones | the value configured on the Garmin device (do not auto-estimate) |
| 6 | Power meter present? Heart-rate strap present? | both assumed; degrade gracefully (N/D) if not |
| 7 | How far back to import at first? (storage has no cap afterwards) | from the first activity with a power meter |
| 8 | Bikes: name, type (MTB/gravel/road/indoor/other), status (active/for sale/sold), purchase date; and which sensors are bike-specific | ask, can be added later in the UI |
| 9 | Preferred default period when the page opens | last 6 months |
| 10 | Pause threshold (days without rides to count as a pause) | 10 |
| 11 | Port and host of the local server | `127.0.0.1:8765` |
| 12 | Colour theme | follow the system, with a sun/moon switch |
| 13 | **Optional** local-AI ride summary (`10`): explain it and ask; default no | no |

State clearly to the user: the route map loads **OpenStreetMap tiles** from the internet (OSM sees the area being viewed, never the ride data; the app falls back to an offline drawing); the Garmin Personal API has **no bike/gear endpoint**; the app contains **no AI model** (the "Coach" is rule-based) unless the user accepts the optional local summary of `10`, which runs on their own computer.

## 5. Architecture and repository layout

Stack (the reference choice; you may substitute equivalents, keeping the same separation of concerns):

- **Backend:** Python 3.12, FastAPI + Uvicorn, `httpx`, `fitparse`, `numpy`, `scipy`, SQLite (stdlib `sqlite3`). Virtual environment in `.venv/`.
- **Frontend:** React 18 + Vite + Recharts, built to static files served by the same FastAPI app (one process, one port).
- **Storage:** `data/<app>.db` (SQLite), `data/fit/<activityId>.fit` (original files), logs in `data/`.
- **Secrets:** `~/.config/garmin/connect-personal-api-key` (mode `600`).

Suggested layout of the app you generate (not of this plan repository):

```text
app/
  backend/            db.py garmin.py sync.py syncall.py fitfull.py power.py derived.py loads.py
                      radar_metrics.py radar_api.py ride_detail.py bikes.py pauses.py wellness.py
                      analytics.py extras.py app.py
  frontend/           index.html package.json vite.config.js src/{App,lib,pages1,pages2,pages3,radar,ride}.jsx styles.css
  data/               <app>.db  fit/  server.log
  config/profile.yaml
  start.sh            start | stop | restart | status | logs | run
```

Architectural principles: one local process; a thin Garmin client; an idempotent, incremental sync; derived metrics stored per ride; analytics endpoints that take a `period` parameter; a frontend that only renders what the API returns.

## 6. Execution phases (build order)

Each phase ends with acceptance checks run on real data. Details are in the file named in brackets.

| Phase | Deliverable | Acceptance (summary) |
|---|---|---|
| 0 | Plan review, questions, `config/profile.yaml`, environment (`.venv`, `npm`) | contradictions listed; answers recorded |
| 1 | Garmin access and first import: activity list (cycling only), details, original FIT files (`01`, `03`) | counts match Garmin; one ride re-fetched is identical; 429 handled |
| 2 | Data model and decoding of every FIT into 1 Hz streams, laps, session, devices (`02`, `03`) | best 20-min power of a ride equals Garmin's own value within rounding |
| 3 | Data cleaning and derived metrics: power spikes, best power, zones, TSS check, climbs, loads (`04`) | spike cases inspected by hand; descent estimate ≈ Garmin elevation gain |
| 4 | Backend API for all pages + sync-on-open (`03`, `06`) | every endpoint returns JSON (never HTML); invalid period → 400 |
| 5 | Frontend shell: four-area menu, period bar, theme switch, shared components (`06`) | all menu entries load with no console errors and no `NaN` text |
| 6 | General statistics pages (`06`, `04`) | each chart has a one-sentence caption, tooltip, table view |
| 7 | Bikes, components and wear, rides-to-match (`05`) | wear numbers match direct SQL totals; alerts at 80 % and 100 % |
| 8 | Wellness: sleep/HRV/resting HR/Body Battery, Recovery and Correlations pages (`03`, `04`) | values match the Garmin API for the same day |
| 9 | Pauses and reasons; Calendar (`06`) | pause bands appear on load/recovery charts and in the calendar |
| 10 | Ride detail, cadence/torque quadrants, aerobic decoupling (`04`, `06`) | route on an OpenStreetMap map with offline fallback; synced charts |
| 11 | Rider profile radar (7 axes, two views) (`04`, `06`) | each axis states its metric, sample size and assumptions |
| 12 | Operations: `start.sh`, incremental-sync test, full-page scan, notes (`07`) | delete one ride from the DB, restart, it is re-downloaded and identical |
| 13 | **Optional**, only if the user said yes: local-AI ride summary (`10`) | with the model server stopped the button shows a readable error; the facts sent contain no GPS or ride name |

## 7. Definition of done

- Every page loads for every period with no console error and no `NaN`/`undefined`/`Infinity` text.
- Opening the app downloads **only** what is new (one list call + new rides + new wellness days).
- Missing data shows N/D and never changes an average.
- Every estimate and every assumption is visible to the user.
- No secret or personal datum is committed or sent anywhere except to Garmin.
- The user can edit profile, bikes, components, thresholds and pause reasons from the UI, and the changes persist.
