# 06 · Pages, navigation and design rules

## 1. Navigation

A left sidebar with **four collapsible areas** (state remembered; the area of the current page is always visible), hash routing (`#page` or `#page/param`, e.g. `#ride/<id>`; a `hashchange` listener must update the view), and a top bar.

| Area | Entry | Uses the period filter |
|---|---|---|
| **General statistics** | Physiology · Volume and zones · Rider profile · Cardiovascular efficiency · Power analytics · Training load · Cadence and torque · Climbs · Evolution · Correlations · Recovery · Stress and sleep | yes |
| | Coach · Data quality | no (fixed windows) |
| **Calendar** | Calendar · Rides to match · Pauses and reasons | no |
| **Single ride** | Ride detail · Browse activities | no / yes |
| **Workshop** | Bike fleet · Components and wear | fleet yes / components no |

**Top bar:** on pages that use the period, the filter row `[ All time | Last 2 years | Last year | Last 6 months | Last 3 months | Last month ]` plus "from <date>"; on the others, a breadcrumb "Area › Page". On the right: sync status text + "Update" button, and a **sun/moon theme switch**; the tab icon is an SVG favicon (`favicon.svg`, in `frontend/public`). **Open on "Last 6 months".** "All time" means everything stored (no age cap, see `01` §5). The filter row scopes **every number, chart and table on the page**; while reloading, keep the previous render dimmed (no skeleton, no layout jump).

Sidebar badges: components over 80 %, rides to match, pauses without a reason. "Ride detail" in the menu reopens the last ride viewed. Every ride title in every table and in the calendar links to `#ride/<id>`.

## 2. Page specifications

### 2.1 Physiology
**Must fit in one screen, no long scroll.** Two rows of tiles, then the identity card. Row one: a single box with distance, climbing and hours (two columns wide), mean NP, FTP, total TSS and TSS/week (with mean IF underneath). Row two, in its own container: the VO₂max gauge and the overnight HRV gauge, each two columns wide, with a "Trend" button that opens a modal with the chart and its own period selector (`04` §15). Give the columns a **maximum width** and leave the free space on the right: the boxes must not stretch with the window. The identity card has the all-time bests at 5 s, 1 min, 5 min, 20 min, 60 min in W and W/kg, one comparison bar per duration against the age and sex table (`04` §15) and a short observation/hypothesis.

### 2.1b Volume and zones
Everything that used to crowd Physiology: weekly hours (toggle km), weekly TSS, ATL/CTL ratio; **peaks in the period (bars) against the all-time best (line with markers)**; EF of rides > 90 min over time; time per power zone and per heart-rate zone (show the 0 W hours separately); table of the **last 15 rides** (date, linked title, km, climbing, mean power, NP, mean HR, EF, TSS — Garmin's, with own value in brackets only if > 2 % apart — and bike with a red "uncertain" pill).

### 2.2 Rider profile
Two cards: the 7-axis radar (view switch *Current form* / *Comparison with types*, a "?" modal with sources and limits, a type menu defaulting to **"closest to you"**) and a table "the seven qualities, one by one" (definition, metric, value, score, sample size) plus the clickable similarity ranking. Open on **Comparison with types**. See `04` §10. In the right-hand table the "measured with" note of each quality is a tooltip behind a "?" next to its name, to keep the table short.

### 2.3 Cardiovascular efficiency
EF per ride (> 90 min), monthly median EF, heart-rate zone time, **aerobic decoupling** chart (dots per ride, zero and median lines) with a sortable table; observation/hypothesis text.

### 2.4 Power analytics
Power-duration curves (two years chosen by the user + the selected period, log time axis, W or W/kg switch), FTP per quarter (estimate vs Garmin), critical power/W′ table (all history and last 12 months), best power per year table.

### 2.5 Training load
PMC (CTL/ATL/TSB, zero line, pause bands), weekly TSS, CTL at week end; caption about the 6-week warm-up.

### 2.6 Cadence and torque
Torque-vs-cadence heat-map for the period (SVG, hover reads the cell: cadence range, torque range, minutes, % of time), quadrant shares table, stacked bars of quadrant share per month, observation/hypothesis/limit text.

### 2.7 Climbs
Scatter (length vs gain, colour = bike, hit area ≥ 24 px) and a sortable table (date, linked ride, bike, km, gain, grade, duration, power, HR, VAM). Caption: algorithm estimate.

### 2.8 Evolution
Two date ranges (defaults per `04` §11), a checkbox "exclude injury/illness pauses and the 14 days after", a table per metric (medians, % change, p, corrected p, plain-language verdict) and the two power curves.

### 2.9 Correlations
A matrix heat-map (diverging blue/grey/red), **a reading guide to the right of the matrix** (rows = columns = the same indicators; colour meaning; strength convention |ρ| < 0.1 negligible, 0.1–0.3 weak, 0.3–0.5 moderate, ≥ 0.5 strong; diagonal; empty cells; p vs ρ; not causation; two examples computed from the data; the hovered cell's values), a toggle to exclude injury/illness days, and a collapsible sorted list **without an inner scrolling area**.

### 2.10 Recovery
Three gauges on top (resting HR, Body Battery maximum, sleep score) against the user's usual range, with a note that it is not a clinical threshold (`04` §15). HRV (night and weekly average), resting HR (+7-day mean), sleep score (+7-day mean), Body Battery (daily max/min), with pause bands. Missing days are empty, never filled.

### 2.10b Stress and sleep
Four tiles (fitness age with real and achievable age, last night's sleep score with Garmin's class, last day's stress with Garmin's qualifier, breathing) and four cards: hours per day at rest and at low, medium and high stress (stacked bars), breathing per minute (mean line with the min–max band), fitness age over time, and a table of last night's sleep items with Garmin's class and optimal range for each. Everything comes from the API (`01`) and is never recomputed; classes are Garmin's, the translation is yours and must say so.

### 2.11 Coach and Data quality
See `04` §13–14.

### 2.12 Calendar
Month grid (Monday first) with navigation (previous/next/today) and a month summary (rides, km, hours, TSS). Each ride is a coloured chip (colour = bike) showing km · hours and TSS (or "no power"), linking to its detail. Pause days are hatched, with "Pause: <reason>" on the first day. A **week-total column** (km, hours, TSS). Legend of bike colours. Hide the week column on narrow screens.

### 2.13 Ride detail
**Five tabs inside the page** (not browser tabs and not separate addresses), in a bar under the sticky header so that it stays in view: *Overview* (KPI tiles, synchronised charts, the bike of the ride), *Route and climbs*, *Pedalling* (torque against cadence on the left, Power phase on the right, in two columns), *Zones and efforts* (power and heart-rate zones, best efforts, aerobic decoupling) and *Laps*. Only the active tab is rendered (the map needs to be visible when it is created); the chosen tab is remembered and stays when moving to the previous or next ride; arrow keys move between tabs. The point is to keep each tab within about one screen.
**Sticky header** (it does not scroll; it sticks just below the always-visible filter bar, on an opaque page-coloured background with a thin bottom border): the previous / next buttons and the ride selector, then the title, then the subtitle (date/time, type, bike pill + uncertain note). The optional "AI summary" button (`10`), if the user chose that feature, sits on the same row, right after "Next". The header must already sit at its sticky position when the page is at the very top (no padding above the filter bar, and cancel the filter bar's bottom margin with a negative top margin), otherwise it visibly "jumps" by a few pixels when scrolling starts. **Previous / Next must not reload the page:** keep the ride page mounted (do not use the ride id in the component key), keep showing the old data slightly faded until the new ride arrives, do not scroll back to the top, and keep the map instance alive (only redraw its overlay). Everything below the header scrolls with the page body: ten tiles (distance & climbing, time & moving time, average speed, mean power & NP, IF · TSS & ride FTP, VI, mean/max HR, EF, cadence, work & temperature).

- **Synchronised charts** (shared cursor via `syncId`): altitude (area), power (with an FTP reference line), heart rate, cadence, speed — **one axis per chart** — with a Time/Distance x-axis switch; each point is the mean of N seconds (state N). Skip the HR chart and print a note when the ride has no HR.
- **Garmin's Training Effect** (aerobic and anaerobic) as a KPI tile, from the FIT session message (`04` §16).
- **Power phase** (right-hand card of the *Pedalling* tab, a tall column: the two rings on top, the PCO under them, then seated and standing times and the pedal values; the PCO explanation is a "?" tooltip; it replaces the older "pedal dynamics" card): one ring per leg, left in green and right in blue, like the power-pedal diagrams: 0° at the top dead centre, clockwise, a light arc for the push phase from start to end and a solid arc for its peak, with the balance percentage in the middle; below, the platform centre offset (PCO, in mm) of each pedal and the seated and standing times. Data from the FIT session message (`avg_left_power_phase`, `avg_left_power_phase_peak`, `avg_left_pco` and the right-hand equivalents). The angles can be stored as text, so parse them; say that the sign of the PCO follows the reference diagram. It also keeps the torque effectiveness and the pedal smoothness (left and right) that the old card showed, and the seated and standing times.
- **Route**: by default a **Leaflet map on OpenStreetMap tiles** (blue polyline from ≤ 600 GPS points, start/end markers, zoom, visible "© OpenStreetMap contributors" attribution, wheel-zoom off so the page still scrolls). A compact toolbar above the map holds the toggle "Map / Offline" and, on the same row, the "Colour by" selector; the choice is remembered; the offline view is an SVG polyline (equirectangular projection scaled by cos(latitude)) with start/end markers and a scale bar, and it is used automatically when tiles fail to load. A layer switcher (top-right icon) offers four base layers, the choice is remembered: **Standard** OSM (`https://tile.openstreetmap.org/{z}/{x}/{y}.png`), **CyclOSM** (cycling, `https://{s}.tile-cyclosm.openstreetmap.fr/cyclosm/{z}/{x}/{y}.png`), **OpenTopoMap** (relief, `https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png`, CC-BY-SA) and **Humanitarian** (`https://{s}.tile.openstreetmap.fr/hot/{z}/{x}/{y}.png`); each layer shows its own attribution. Do not add layers that need an API key or forbid this kind of use (commercial satellite imagery, OpenCycleMap/Thunderforest without the user's own key). Create the map **once** and redraw only the overlay when data or colour change (the ride payload is reloaded while a sync runs: recreating the map would reset the zoom); use `detectRetina` for sharp tiles on high-density screens, half-step zoom, and enable wheel zoom only after a click so the page still scrolls. A "⤢ Enlarge" button next to the selectors opens a **wide modal** (about 96 % of the width and 92 % of the height) with the same map, the "Colour by" selector, the legend and the layer switcher; Esc or a click outside closes it. A note under the map says that the tile server sees the viewed area. **Coloured track:** the "Colour by" selector (solid / speed / gradient / altitude / power / heart rate / cadence; options without data are hidden, and the choice is remembered). The API returns per-point values: means over ±10 s (altitude ±3 s) and the gradient over ±20 s, set to N/D if less than 40 m were covered in that stretch. Colour scale: sequential violet → yellow between the 5th and 95th percentile of the ride (so spikes do not flatten it); the gradient uses a diverging scale (blue descent, grey flat, red climb) symmetric around 0. A "Gradient" selector (remembered) offers four palettes: **Contrast** (blue → violet → red → orange, the default), **Warm** (violet → orange), **Blue** (light → dark) and **Viridis**; the first three avoid very light colours (they vanish on pale roads and meadows) and green (it clashes with woods on the map), and each has a diverging variant for the gradient with a dark-grey midpoint. Draw a dark semi-transparent casing under the coloured line so every colour separates from the background, and show a gradient legend with the min and max values and units.
- Torque-vs-cadence plot with the ride's quadrant shares and median cadence/torque.
- Aerobic decoupling card (or the reason it is unavailable); pedal-dynamics card (L/R balance, torque effectiveness, smoothness, time standing).
- Time in power and HR zones; best efforts of the ride (5 s … 60 min) with their **rank among all rides** and the all-time best; detected climbs; laps table (km, time, speed, power, NP, max, HR, cadence, ascent, descent); bike selector with "Confirm".

### 2.14 Browse activities
Search box and sortable columns (same columns as the last-15 table), inline bike selector, red border on uncertain rows.

### 2.15 Rides to match, Pauses, Bike fleet, Components
See `05` and `04` §12. Each bike card also shows the four value figures of `05` §5 (list price, total cost, value with upgrades, estimated resale as a range), and the fleet page reloads the bikes every time it opens. The Pauses page lists the automatic pauses (dates, days, last ride → next ride, a reason selector, a note, Save) plus a form to add a shorter manual pause, and explains exactly what the reason does and does not change.

### 2.16 User profile
A window opened by a **profile icon next to the app title** (not a menu page) is the only place for the user's personal input, which lives in the database and is **never written in the code** (`02`, table `athlete`): sex, birth date (age is computed from it), weight, maximum heart rate, and the mass of bike plus gear used by the braking-load estimate. A button reads the maximum heart rate, the five zone floors and the lactate-threshold heart rate from Garmin's heart-rate zones (read-only); changing the maximum heart rate by hand discards the zones read from Garmin. A second button recomputes the derived data that depend on the profile (time in heart-rate zones, braking load). Validate every field (weight 30–200 kg, maximum heart rate 120–230, birth date not in the future) with a readable message. A missing value gives N/D wherever it is needed, never a guess.

## 3. Design system and chart rules

- **One axis per chart.** Never a dual-axis chart: two measures → two charts, small multiples or normalised.
- **Colour is a job:** categorical (identity, fixed order, never cycled; a filter must not repaint surviving series), sequential (one hue light→dark, for ordered magnitudes such as zones), diverging (two hues + neutral grey midpoint), status (reserved; always with icon + label). Text never wears a series colour: use ink tokens; a coloured mark beside the text carries identity.
- **Marks:** bars ≤ 24 px with a 4 px rounded data end and square baseline; lines 2 px; markers ≥ 8 px with a 2 px surface-colour ring; areas at ~10–14 % opacity; hairline solid gridlines; a 2 px surface-coloured gap between stacked segments (no outlines).
- **Legends:** always for ≥ 2 series; none for one. Direct labels sparingly, never on every point.
- **Tooltips:** hover **and** keyboard focus; one tooltip listing every series at the cursor, value first (bold), series name second, line-shaped key; labels inserted as text (never as HTML); hit areas ≥ 24 px for dots.
- **Every chart has a one-sentence caption and a "Table" view toggle** (the same data as a sortable table).
- **Disable chart entry animations** (they slow 700-point lines and make screenshots/tests unreliable).
- **Light/dark:** define all colours as CSS variables, with the dark set both under `prefers-color-scheme` and under an explicit `data-theme` attribute; the switch stores the choice. Sequential ramps for zones differ per mode (in dark mode higher intensity = lighter).
- **Accessibility:** buttons are real `<button>`s with `aria-expanded`/`aria-pressed`/`aria-sort`; modals use `role="dialog"`, `aria-modal`, focus the dialog on open, close on Esc/outside click/✕ and restore focus; icons have text alternatives; tables have real headers; do not rely on colour alone.
- **Modals instead of long inline blocks** for explanations ("?"), add/edit forms and glossaries. **No inner scrollbars** inside cards when the page already scrolls (they fight the page scroll); use inner scrolling only for genuinely long tables with sticky headers.
- **Cards that hold SVG/charts must grow with their content**: a fixed height clipped a 9×9 matrix once.
- **Empty states** always say why and what to do ("No rides to match: all rides have a confirmed bike…").

## 4. Frontend engineering rules (pitfalls met in the reference build)

- React hooks must run in the same order on every render: put all `useState` before any early `return`.
- In JSX text, escape `<` (write `&lt;`); a bare `<30°` breaks the build.
- Do not define components inside other components if they hold state.
- `Recharts`: category x-axis reference areas need x values that exist in the data (snap to the nearest day); `syncId` shares the cursor across stacked charts; hide inner x-axis labels except the last chart.
- Round-trip every edit with a **full-page scan** (open every page for at least two periods, assert no console error, no `NaN`/`undefined`/`Infinity` in the text, no "Loading" stuck). A page that silently shows "Loading…" means the endpoint returned HTML (see `07`).
- When a user-facing label changes, search the source for the old text, the docstrings and the project notes.
- Persist small UI preferences in `localStorage` (theme, closed menu groups, accordion state, table sort, last ride).
