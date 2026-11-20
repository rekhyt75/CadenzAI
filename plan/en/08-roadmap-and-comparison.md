# 08 · Comparison with GoldenCheetah, roadmap and non-goals

This section records why the reference app contains what it contains. It compares it with **GoldenCheetah** (open-source desktop cycling analytics, GPL-3.0; v3.8 was released on 20 Sep 2026). The comparison is based on its README, release notes and user-guide pages, **not on hands-on use**; the full metric glossary was not read. Re-check before relying on it.

## 1. Where each one is stronger

| Area | GoldenCheetah | This app |
|---|---|---|
| Data in | file import and cloud services (Strava, Tredict, Withings, TodaysPlan…); no direct Garmin Connect link found in the sources read | automatic, incremental sync from the Garmin Personal API on every start; original FIT files kept |
| Load | PMC with several stress metrics (BikeStress, TRIMP, RPE), Banister model, "expected PMC" from planned workouts | CTL/ATL/TSB on Garmin's TSS, ATL/CTL ratio, pause bands |
| Power curve | mean-max with critical-power models, W′ balance | per-year/period curves, 3-point CP, FTP estimate |
| Single ride | full activity chart, map, histograms, intervals, force/velocity quadrants, HR vs power, CdA estimation | ride detail page (synchronised charts, route, laps, zones, best efforts, torque quadrants, decoupling) — built after the comparison |
| Pedal dynamics | quadrant analysis | stored and shown per ride and as averages |
| Data cleaning | ~20 processors (power/HR spikes, GPS, gaps, freewheeling duplicates, elevation…) | power-spike filter, quality page; HR spikes and GPS fixes **not** built |
| Recovery | HRV and weight via external services | sleep, HRV, resting HR, Body Battery from Garmin, correlated with load; recovery axis in the radar |
| Equipment | tracks equipment use | per-bike components with thresholds **and** a route-aware usage index, alerts, sensor-based bike matching |
| Statistics | R/Python scripting | built-in period comparison with a test and correlations |
| Planning | calendar, plans, workouts, ERG trainer control (v3.8 Plan view) | none |
| Extensibility | user-defined metrics, R, Python | the code itself |

The "freewheeling duplicate" processor of GoldenCheetah removes the same artefact as the power-spike filter in `04` §1.1: an independent confirmation of the approach.

## 2. Roadmap (candidates, in value order)

1. **W′ balance** with a better CP model — relevant for XC/MTB repeated surges; only trustworthy if CP/W′ come from real maximal efforts (offer a guided test or a ramp test import), otherwise label it clearly as an estimate.
2. **Heart-rate spike and GPS cleaning**, plus a "fix and recompute" action in Data quality.
3. **Export** of tables to CSV and a one-command backup of the database and FIT folder.
4. **Advanced filters** in Browse activities (bike, type, date range, power/HR present).
5. **Weather/temperature effect** on EF (temperature is in the stream) to explain EF dips.
6. **Planning** (calendar of planned rides and expected load) — only if the user asks for it.
7. **Import of pre-power-meter history** (older rides with power metrics N/D) — only if requested.
8. **Internationalisation:** a string dictionary and a language switch.

## 3. Deliberately not built

Trainer/ERG control and workout editor, running and multi-sport analytics, user formula editors, R/Python runtimes, many import formats, upload to Strava or other services, aerodynamic CdA estimation (little value at MTB speeds — untested), any claim of "AI" (the Coach is rule-based).

## 4. Honest limits to repeat to the user

- Percentile references are a population table, not age- or discipline-corrected; best efforts in rides are not tests, so threshold, VO₂max and critical power are probably under-estimated.
- The type shapes in the radar and the wear thresholds are the author's hypotheses.
- The Garmin Personal API does not expose bikes; the bike matching is inference.
- The usage index and braking load come from a physical model with declared assumptions.
- Sample sizes in short periods can be tiny (e.g. a dozen long rides, a handful of hard days); show them.
