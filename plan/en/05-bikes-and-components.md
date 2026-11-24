# 05 · Bikes, automatic bike matching, components and wear

## 1. Bike fleet

A `bikes` table the user edits from the UI (never hard-code bikes). Fields: `name` (unique), `kind` (`mtb`, `gravel`, `road`, `indoor`, `other`), `status` (`active`, `for sale`, `sold`), `bought_on`, `sold_on` (optional), `price` (purchase), `list_price`, `sold_price`, `notes`, `color_slot`.

- **Colour is an identity:** each bike gets a categorical colour slot (1–8) when created and **keeps it forever**; adding or selling a bike must never repaint the others. Use the validated categorical order (see `09`).
- **Add / edit / delete:** a "+ Add bike" button (Workshop → Bike fleet and Components) opens a modal. A bike with rides or components cannot be deleted (the server answers 400 with a clear message): the user marks it **sold** instead. Renaming a bike updates its rides and components.
- A **sold** bike is hidden/collapsed by default in the components page but its history stays.
- The bike fleet page shows per bike: rides, km, hours, climbing, rides with a bike to confirm; a monthly stacked-bar of km per bike; status pills.

## 2. Assigning a bike to every ride

The Garmin Personal API does **not** expose which bike a ride was recorded on (see `01`). Rules:

**Historical rides** (from the data export, `*_gear.json`): use Garmin's assignment. If the user has a rule that disagrees with Garmin for some rides (forgot to sync the gear, etc.), let the **user decide**: flag those rides as *uncertain* and show them with a red border until confirmed.

**New rides** (every new ride must end up with a bike; automatic, in this order):

1. **Sensor evidence (certain).** Build, from rides whose bike is reliable (Garmin's or user-confirmed), the set of external sensors (manufacturer + serial; ignore the heart-rate strap, which is worn by the rider, and the head unit) and keep those seen on **exactly one bike** in **at least 3 rides**. If a new ride contains such sensors and they all point to one bike → assign it, `bike_uncertain = 0`, and write the reason in `bike_note` ("recognised by sensor X, seen only on this bike in N rides"). If the indicated bike is sold or not yet bought at the ride date, assign it but flag it uncertain. Shared sensors (a power meter moved between bikes) carry no information.
2. **Ride type and bikes in use (to confirm).** Among bikes of the matching kind (`gravel_cycling`→gravel, `mountain_biking`→mtb, `road_biking`/`cycling`→road, `indoor_cycling`/`virtual_ride`→indoor) that were already owned at the ride date and not sold, take the most recently bought. Flag uncertain.
3. **Fallback (to confirm):** the not-sold bike used most recently. Never leave a ride without a bike if at least one bike exists.

A ride the user assigns or confirms by hand sets `bike_manual = 1` and is **never overwritten again**. Re-running the assignment only touches rides **without** a bike.

**Validation of rule 1:** leave-one-out over the history (recompute the sensor evidence without the ride being tested): in the reference build 97 of 97 rides carrying a bike-exclusive sensor were identified correctly.

## 3. "Rides to match" page

Lists rides with `bike_uncertain = 1` (or no bike): date, linked title, type, km, **why** it needs confirmation, a bike selector and a "Confirm" button. Empty state: a short message. The sidebar shows a count badge. Confirming is instant and persistent.

## 4. Components and wear

### 4.1 What the user registers (per bike)

`kind` (chain, cassette, chainring, front/rear brake pads, front/rear discs, front/rear tyre, sealant, front/rear brake fluid, fork lower-leg service, fork full service, shock air service, shock full service, plus "other" with a free name), `model`, `installed_on`, `price`, `vendor`, `notes`, up to four thresholds (`km`, `moving hours`, `climbing m`, `days`) and an optional `removed_on`. Persist them in the DB; show N/D for unknown price/vendor (never invent).

**Front/rear are separate** for pads, discs, tyres and brake fluid.

### 4.2 Registering a new bike's components from its spec sheet

When the user names a bike model, look up the **manufacturer's spec sheet** and pre-fill every component at the **purchase date** (frame, fork, shock, drivetrain, brakes, rotors, wheels, tyres, cockpit, seatpost, saddle…), marking items with no wear threshold as "registered without threshold". Rules:

- Prefer the manufacturer's page; **do not circumvent bot protection**. If it is blocked, say so; ask the user to open the page in their browser (then read that tab) or use two independent public retailers/databases and cross-check them.
- Record the `source` of every model string. If the buyer's dealer changed a part (e.g. wheels), the **user's statement wins** and the source says so.
- Never put an unverified model in a component row; write "not in the spec sheet" instead.

### 4.3 Thresholds are estimates

Offer a template of **indicative** thresholds (state that they are **not manufacturer data**) and let the user edit them. Reference values used by the reference build: chain 2 500 km, cassette 7 500 km, chainring 10 000 km, front pads 1 800 km, rear pads 2 800 km, front disc 8 000 km, rear disc 10 000 km, front tyre 3 000 km, rear tyre 2 200 km (light XC rubber), sealant 90 days, front fluid 540 days, rear fluid 730 days, fork lower-leg service **100 h or 365 days** (whichever first), fork full service 730 days, shock air service 100 h or 365 days, shock full service 730 days. Cite where intervals come from and how weak the source is (e.g. an old manufacturer table plus a dealer's page); the first version of the reference build guessed 50 h for suspension and produced an alarm after six weeks: avoid unsupported figures.

### 4.4 Two wear indicators

For each component, since its installation date, using only rides of **that bike**:

1. **Wear on the threshold** (as set): the **maximum** over the defined thresholds of `used / threshold`, where used = km, **moving hours** (not elapsed time: stops do not move a suspension), climbing, days.
2. **Usage index (route-aware):** the same consumption weighted by what the route actually did to the part. The threshold is the component's life **at the rider's average mix of rides**; a ride with twice the braking per km uses brakes twice as fast.

```text
index = mean over drivers d of  Σ_since_install(driver_d) / ( ref_d · threshold )
ref_d = (Σ over the whole history of driver_d) / (Σ over the whole history of the threshold unit)
threshold unit priority: km, else moving hours, else days (days use the history span)
```

Drivers (declare them in the UI; they are modelling choices):

| Component | Driver(s) |
|---|---|
| chain, cassette, chainring | pedalling work (kJ) |
| pads, discs, brake fluid | estimated braking load (kJ, see `04` §7) |
| front tyre | km and braking load |
| rear tyre | km, pedalling work and braking load |
| fork, shock | moving hours and hours spent descending |
| sealant, wheels, derailleur, cockpit, etc. | none (time only, or not monitored) |

The braking load is **not split front/rear**; the difference between front and rear parts lives in their thresholds. Components without a route-dependent driver show "not applicable".

### 4.5 Alerts

A component is "watch" at **≥ 80 %** and "over" at **≥ 100 %** of the **higher** of the two indicators; say which one triggered. Show: a meter with an icon and a text label (never colour alone: ✓ within limit, ▲ over 80 %, ● over 100 %), a banner on top of the page, and a badge in the menu.

### 4.6 Page behaviour

- One **accordion section per bike** (state remembered), with the bike name, a **"+" button right of the name** that opens an *Add component* modal (type from the template list, model, installation date, price, vendor, thresholds, notes), a status pill and an "Edit bike" button.
- Each table is **sortable by column**; the **default sort is most worn first** (descending maximum of the two indicators; components without a value last; removed ones always at the bottom; the choice is remembered, with a "restore" button).
- A **"?" help modal** (not a block at the top of the page) explains the two indicators, the drivers, the assumptions (mass, Crr, CdA), and a short glossary (lower-leg service, air-sleeve service, full service).
- **Factory-fit components** (flag `original`) have no price or vendor: they are part of the bike's price and are never counted as "missing price".
- **Removing a component asks what to do** (button "Remove"): *replaced* (it stays in the maintenance history with its date; it no longer counts for alerts or resale, and its cost stays in the total cost) or *delete for good* (a test or a mistake: it disappears and its price leaves the total cost). A replaced part offers only "Put back" and "Delete".
- An **"Edit" modal** per component (model, install date, price, vendor, "is an upgrade" tick and added value (`is_upgrade`, `upgrade_value`), thresholds, notes, mark removed, delete with confirmation).
- Updating happens automatically: wear is **computed at request time** from the rides of the bike since installation, so a new ride (or a corrected bike assignment) is reflected immediately. **Test:** remove one ride → numbers drop; re-sync → they return exactly.

## 5. Bike value: four distinct figures

Each bike card on the fleet page shows four figures (a **"?"** button explains them). Missing data gives **N/D** and says what is missing, never an invented number.

| Figure | Definition |
|---|---|
| List price | the new price of the base model (`list_price`, entered by the user) |
| Total cost | purchase price **plus every amount spent on components** (consumables, services, upgrades). It only grows: it is what the bike cost over time |
| Value as new with upgrades | list price plus the added value of the **upgrades** fitted and not removed. Consumables (pads, tyres, chain) do **not** raise it: nobody sells a bike without its pads |
| Estimated resale | list price × the share of value retained at the bike's age, plus the upgrades, shown as a **min–max range** (the large number is the centre). A sold bike shows the real `sold_price` when entered |

A component is an upgrade only if the user ticks **"This is an upgrade"** (`is_upgrade`); its optional `upgrade_value` (empty = its price) is the value it adds as new.

**Depreciation model: use a sourced one, and say where it comes from.** Verified source: buycycle, "Quanto posso vendere la mia bici?" (`https://buycycle.com/blog/it/post/quanto-posso-vendere-la-mia-bici/`). It gives value retained on the list price by age band, and the rule "20–40 % lost in the first year, then 10–15 % a year":

| Age | Retained |
|---|---|
| under 1 year | 70–80 % |
| 1–2 years | 60–70 % |
| 3–4 years | 45–60 % |
| 5–6 years | 35–50 % |
| 7 years or more | 25–40 % |

Declare what is **your** assumption and not the source's: each band applies at the middle of its age range with a straight line between points; beyond 8 years −12 % a year with a floor of 15 %. The source does **not** split MTB, gravel and road, and gives **no percentage for upgrades**: a recovery such as 20–40 % of the upgrade's value is an unverified assumption, so label it as such in the UI, or ask the user. Do not use unsourced figures from an assistant's answer as if they were data: open the page and check. Ignore mileage, condition, frame size and local market, and say so. The real list price should be the street price at the time of sale.

**Worn parts lower the resale a little.** A buyer deducts the cost of replacing what is worn, even though a new chain would not raise the value. For each priced, not removed component deduct its price in proportion to the wear: nothing up to 80 % wear, a straight line up to the full price at 100 %. This rule is your assumption, not the source's: say so. Factory-fit parts have no price, so if they are worn only report it and deduct nothing.
