# 04 · Metrics, formulas and validation

Every formula below states its **parameters**, whether it is a **measurement or an estimate**, and a **check** you must run on real data. Do not ship a metric whose check you could not run: report it as unverified.

## 1. Stream preparation

Build a regular **1 Hz grid** over the ride (`n = last_t + 1` samples): fill each channel at its timestamp, leave the rest `NaN`. A pause is missing data, not zero.

### 1.1 Power spike removal (measurement hygiene)

Power meters occasionally report a value held for 1–2 s while the bike is stopped or freewheeling (e.g. 2 000–4 000 W with cadence 0). Treat as missing a **group** of samples when **all** hold:

- consecutive samples with `power > 0` and `cadence == 0`;
- group length ≤ 5 s;
- the second before the group and the second after have no power (zero or missing);
- the group's maximum is ≥ **400 W** (lower groups cannot distort a peak, so leave them).

Set them to `NaN` (never to 0). Record the count and max per ride in `power_quality`.

Do **not** require speed = 0: the same artefact appears while the bike is pushed slowly. Do **not** drop real standing starts: they have continuous power, not an isolated group. **Check:** the best 1 s and best 5 s over all rides must form a smooth curve (e.g. 1 s ≈ 900 W, 5 s ≈ 850, 15 s ≈ 790); a best 1 s above ~2 000 W means a spike survived. Inspect the largest removed groups by hand (same value repeated 2 s, zeros around).

### 1.2 Best power per duration

For durations `[1, 5, 10, 15, 30, 60, 120, 300, 600, 1200, 1800, 3600, 5400]` s compute the maximum rolling mean on the grid, accepting a window only if **≥ 95 %** of its seconds have power. Store per ride in `best_power`. **Check:** the best 20 min of a ride equals Garmin's own "max 20 min power" (`summaryDTO.maxPowerTwentyMinutes`) within rounding.

### 1.3 Normalised power (NP)

30 s rolling mean (zeros for missing), 4th power, mean, 4th root. Use Garmin's `normPower` as the stored value and your own `NP()` only for sub-ride splits (halves, laps).

## 2. Load and intensity

- **TSS (main figure): Garmin's `trainingStressScore`.** Standard formula for the check: `tss_own = duration_s · (NP/FTP)² / 3600 · 100`, with the ride's own FTP. Show your value **only if it differs by more than 2 %** from Garmin's (then show both, Garmin first). In the reference build ~77 % of rides agreed within 2 % and the rest differed by up to ~19 % for a reason that could not be identified: say so, do not pick a winner.
- **IF** = NP/FTP (Garmin's `intensityFactor` agreed in 100 % of rides).
- **Variability Index** = NP / average power.
- **PMC** (only rides with measured power, Garmin TSS): `CTL += (TSS − CTL)/42`, `ATL += (TSS − ATL)/7`, `TSB = CTL(yesterday) − ATL(yesterday)`; one value per calendar day starting at **0 on the first ride**. The first 42 days under-estimate CTL: show the ATL/CTL ratio only after day 42 and write the caveat on the page. TSS per day is summed over the day's rides.
- **Weekly volume:** ISO weeks (Monday start): hours, km, climbing, TSS, ride count.
- **KPIs for a period:** km, climbing, hours, mean NP, total TSS and TSS per week (total / weeks in the period), mean IF, ride counts with power and with HR.

## 3. Zones

- **Power zones (Coggan, % of the ride's FTP):** Z1 < 55 %, Z2 ≤ 75 %, Z3 ≤ 90 %, Z4 ≤ 105 %, Z5 ≤ 120 %, Z6 ≤ 150 %, Z7 > 150 %. **0 W is a separate bucket** ("not pedalling", important in MTB). Count valid seconds.
- **Heart-rate zones:** the user's own zone floors from the device (cycling profile) with their max HR; keep a "below Z1" bucket. **Do not recompute max HR**: if observed peaks exceed the configured max (artefacts or a mis-set max), list those rides in Data quality and leave the zones alone.
- Aggregate zones over a period as **time**, and say how many rides contributed.

## 4. Efficiency

- **EF** = NP / average HR, rides **> 90 min** with both channels. Show per ride and monthly medians.
- **Aerobic decoupling (Pw:Hr), per ride and trend:** take the **moving samples only** (speed > 1 m/s), split them in two halves by count, compute `EF = NP(half) / mean HR(half)` for each, `decoupling % = (EF1 − EF2)/EF1 · 100`. Require ≥ 3 600 moving samples and HR present in ≥ 80 % of them. Positive = heart rate drifted up at the same output. **Do not split by wall-clock time**: long stops produce absurd values (a −97 % case disappeared after switching to moving samples; reference range afterwards −13 … +23 %, median ≈ 10 %). Caption: indicative; terrain makes it noisy; a "< 5 % is good" rule of thumb exists for steady efforts but is not validated for MTB.

## 5. Power analytics

- **Power-duration curve** per calendar year, for the selected period and for any two years chosen by the user; log-scale duration axis; optional W/kg.
- **FTP estimate** per quarter = **95 % of the best 20 min** of that quarter (an estimate, **not a test**); show next to the FTP stored by Garmin and say neither is a test.
- **Critical power (CP) and W′**: linear work-time regression on the best **5, 10, 20 min** efforts (`work = CP·t + W′`). An estimate: efforts in rides are not maximal tests, so CP and W′ are probably **under-estimated**. Show for the whole history and the last 12 months.
- Weight for W/kg: from the profile (default: latest Garmin value, noting if it is a manual profile entry).

## 6. Climbs (estimate)

Smooth altitude with a 15 s moving mean. Walk the series; a climb starts at a local minimum and continues while the climb stays within **12 m of its running peak**; keep it if **gain ≥ 40 m, length ≥ 400 m, mean grade ≥ 3 %**. Store start, duration, distance, gain, grade, mean power, mean HR, VAM (`gain/duration·3600`), start coordinates. **Check:** per ride the sum of climb gains must not exceed the ride's total climbing (reference: max 95 %, mean 65 %). It is an estimate with fixed thresholds: say so.

## 7. Wear loads (estimates) — braking, pedalling work, descent

Physical energy balance per second, on a 1 Hz grid with altitude smoothed over 15 s and speed over 3 s:

```text
P_brake = P_gravity + P_pedal − dKE/dt − P_rolling − P_aero      (clipped to 0 … 2000 W)
P_gravity = −m·g·d(altitude)/dt        P_pedal = cleaned power (0 if missing)
dKE/dt    = m·v·dv/dt                  P_rolling = Crr·m·g·v      P_aero = 0.5·ρ·CdA·v³
counted only when speed > 1.5 m/s and data are valid
```

Parameters (declare them as assumptions in the UI): **m = rider + bike + kit** (reference 95 kg), **Crr = 0.012**, **CdA = 0.5 m²**, **ρ = 1.2 kg/m³**, g = 9.81. Per ride store: `brake_kj = ΣP_brake/1000`, `descent_m` (sum of negative altitude steps of the smoothed series), `descent_h` (hours with grade < −3 % and speed > 1 m/s), `pedal_kj = Σpower/1000`, moving hours.

**Checks (all three must pass):** (1) estimated descent / Garmin's elevation gain ≈ 1 on closed loops (reference median 0.99); (2) `brake_kj` / potential energy lost `m·g·descent` is between 0 and 1 (reference median 0.6); (3) flat gravel rides show a tiny braking load per km and steep descents a very large one (reference ratio ~80×). Rank correlation between `brake_kj` and `descent_m` > 0.9.

## 8. Torque vs cadence (quadrants)

`torque (Nm) = power / (cadence · 2π/60)` for samples with power > 0 and cadence ≥ 20 rpm and torque ≤ 100 Nm. Histogram in seconds: cadence bins of 5 rpm (0–140), torque bins of 2.5 Nm (0–100); store sparse. Plot as a heat-map with constant-power curves (100/200/300/400 W). **Reference lines (a convention, state it in the UI):** cadence **80 rpm**, and the torque that gives the user's **FTP at 80 rpm** (`FTP·60/(2π·80)`). Quadrants: high torque/high cadence ("hard effort"), high torque/low cadence ("force"), low/low ("easy and slow"), low torque/high cadence ("agile"). Report time share per quadrant, per ride, per period and per month. Interpret cautiously: terrain dictates cadence in MTB.

## 9. Pedal dynamics

From the session message when a dual-sided meter is used: left/right balance (decode as in `02`), torque effectiveness and pedal smoothness per side, time standing. Present as averages per ride and as a trend; do not over-interpret small asymmetries (a ~1–2 point offset is normal) and compare periods before/after an injury only as an observation.

## 10. Rider profile radar (7 axes) — all proxies

| Axis | Metric (proxy) | Score for the "types" view |
|---|---|---|
| VO₂max / aerobic capacity | best 5 min power, W/kg | percentile-equivalent in the reference table (`09`) |
| Threshold (FTP) | best 20 min power, W/kg | same, 20 min column |
| Sprint / anaerobic | mean of best 5 s and best 1 min, W/kg | mean of the two percentile-equivalents |
| Endurance | best 20 min **after ≥ 1 500 kJ of work** in the same ride, as % of the fresh best 20 min | **own threshold:** retention 70 % → 0, 100 % → 100 |
| Climbing | best mean power on a detected climb ≥ 10 min, W/kg | percentile-equivalent, 20 min column (approximation) |
| Flat / steady pace | best 20 min on a near-flat stretch (net altitude change ≤ 1.5 % of distance, ≥ 3 km, little up-and-down), W/kg | same |
| Recovery | days to bring night HRV back to the lower limit of the user's baseline band after a day with TSS ≥ 250 (within 7 days, ≥ 1 usable HRV record) | **own threshold:** 1 day → 100, 5 days → 0 |

Percentile-equivalent: linear interpolation between the published tiers (30/55/75/90/98); **below the 30th percentile there is no data, so draw a straight line from 0** and say it is an assumption. Always show sample sizes (rides, climbs, hard days) and warn when small.

Two views behind a switch:

1. **Current form:** each axis as % of the user's **own best over the whole history** (100 %) for the selected period (for recovery 100 % = the user's historical mean; more days = lower). Ring at 100 %. With the period "All time" everything is 100 %: show a note.
2. **Comparison with types:** the 0–100 scores of the whole history, plus an optional overlay of a rider type's **shape** brought to the user's mean level. The type shapes are the author's **qualitative hypotheses, not measurements** (weights per axis: sprinter, puncheur, MTB/XC, climber, time-trialist/rouleur, endurance/gran-fondo, all-rounder) — state it prominently and let the user edit them. The **closest type** (Euclidean distance between mean-normalised shapes) is recomputed on every page load on the **whole history**, independent of the period filter; preselect it, show a clickable similarity ranking (100 = closest of the set, 0 = farthest) and explain that it is not fixed.

## 11. Statistics for comparisons and correlations

- **Evolution (compare two periods):** per ride metrics (NP, mean power, IF, EF, TSS, km, climbing, hours); median of each period, % change, **Mann-Whitney U two-sided** with at least 5 rides per period, and **Bonferroni** correction for the 8 metrics. Say that rides are not independent. Default: selected period vs the previous period of equal length; for "All time", first half vs second half of the history. Optional exclusion of injury/illness pauses + the 14 days after (see below).
- **Correlations (Spearman):** daily series of previous-day TSS, previous-7-day TSS, CTL, TSB, resting HR, night HRV, sleep score, daily Body Battery maximum, stress. Skip pairs with < 30 common days. **Exclude pairs among the load metrics themselves** (they are derived from the same series). Optionally exclude days inside injury/illness pauses. Always show ρ, p and n; warn that it is not causation.

## 12. Pauses

A **pause** = at least *N* consecutive days (default 10) between two rides (also an ongoing one after the last ride). Pauses are computed from the rides; only the **reason** (injury, illness, travel, work, planned rest, weather/maintenance, other) and a note are stored. The reason **never changes any computed value** (TSS, CTL…): after a stop CTL must fall regardless. Reasons *injury* and *illness* can optionally exclude the pause plus the following **14 days** from period comparisons and the pause days from correlations. Draw pauses as grey bands on the load and recovery charts and as hatched cells in the calendar.

## 13. Coach (rule-based)

A handful of cards, each with **Observation / Hypothesis / Recommendation** written from computed numbers: load and freshness (CTL 6 weeks ago vs now, TSB), intensity distribution (share in Z1–Z2 / Z3 / Z4+), EF trend with its p-value, pauses and returns (with their reasons), recovery (7-day vs 60-day HRV and resting HR), data quality of recent rides. No medical advice. No AI model is involved unless the user adds one.

## 14. Data quality page

Counts and lists of: rides without power / without HR (by month), discarded power samples, rides where own TSS differs > 2 %, heart-rate peaks above the configured max, bikes to confirm, pauses without a reason, days missing sleep/HRV/resting HR (computed from the actual span of watch data), plus the list of **known limits** (weight is a profile value; FTP estimate is not a test; climbs are an algorithm; CTL warm-up; the API does not expose gear).
