# Railway Station Operations & Passenger Analytics — Complete Project Guide

**Goal:** Recreate the 4-page reference dashboard (Executive Overview, Station Operations,
Revenue Analytics, Passenger & Punctuality) in Tableau, using the 5 CSVs provided, while
deliberately practicing LOD expressions, parameters, calculated fields, table calcs,
dashboard actions, and design polish — the things that make a mentor stop scrolling.

Data window: **1 Jan 2024 → 17 Jul 2026** (today), so `TODAY()` / `DATEDIFF` / relative date
filters all work correctly. Three full-ish years = clean YoY and trend behaviour.

---

## 0. Files You Have

| # | File | Grain (1 row =) | Rows |
|---|------|------------------|------|
| 1 | `dim_stations.csv` | one station | 56 |
| 2 | `dim_trains.csv` | one train | 54 |
| 3 | `dim_calendar.csv` | one calendar date | 1,096 |
| 4 | `fact_train_bookings.csv` | one train + one travel date + one class of travel | 122,219 |
| 5 | `fact_station_daily_ops.csv` | one station + one date | 52,024 |

Total ~174K rows. `fact_train_bookings` alone clears 1 lakh — this is your primary fact
table and drives 3 of the 4 pages. `fact_station_daily_ops` drives the Station Operations
page. Everything else is a lookup ("dimension") table.

This is a deliberate **star schema** — one central date/train/station skeleton, two facts
hanging off it. Nothing snowflakes further, nothing needs a 6-table blend. That's what makes
it "simple to connect."

---

## 1. Data Model (how the 5 files relate)

```
                     dim_calendar (date)
                      /                \
                     /                  \
        fact_train_bookings        fact_station_daily_ops
         (travel_date)                   (date)
          |         |                       |
    dim_trains   dim_stations  ←────────────┘
   (train_id)   (station_id, used twice in bookings:
                 source_station_id & destination_station_id)
```

**Relationships to build in Tableau (Data Source tab → use "Relationships", NOT legacy joins):**

| From | Field | To | Field | Type |
|---|---|---|---|---|
| fact_train_bookings | `train_id` | dim_trains | `train_id` | many-to-one |
| fact_train_bookings | `source_station_id` | dim_stations | `station_id` | many-to-one |
| fact_train_bookings | `destination_station_id` | dim_stations | `station_id` | many-to-one* |
| fact_train_bookings | `travel_date` | dim_calendar | `date` | many-to-one |
| fact_station_daily_ops | `station_id` | dim_stations | `station_id` | many-to-one |
| fact_station_daily_ops | `date` | dim_calendar | `date` | many-to-one |

\* Tableau relationships only let you link one field pair per relationship line. Since
`dim_stations` needs to join on **both** `source_station_id` and `destination_station_id`,
bring `dim_stations` into the canvas **twice** — rename the second instance
`dim_stations (Destination)`. This is the one slightly non-obvious step; everything else is
a straight drag-and-drop.

**Why Relationships instead of Joins?** Relationships keep each table at its native grain
(no row duplication) and let Tableau auto-pick the right level of aggregation depending on
which sheet you're on — e.g., station-level cleanliness won't get inflated just because you
also dropped a booking-level measure on the same viz. This alone is worth a line in your
presentation: *"Used Tableau's relationship model instead of joins to avoid fan-out/
duplication between the two fact tables."* Mentors love hearing that sentence.

**Do not join `fact_train_bookings` to `fact_station_daily_ops` directly.** They're
different grains (train-level vs station-level) and both link back through
`dim_calendar`/`dim_stations` — that's the whole point of a star schema.

---

## 2. Data Dictionary

### 2.1 `dim_stations.csv` — Station Master

| Column | Type | Description |
|---|---|---|
| `station_id` | String (PK) | Unique station key, e.g. `ST001`. Used by both fact tables. |
| `station_name` | String | Full station name (e.g. "New Delhi", "Mumbai CSMT"). |
| `station_code` | String | Short station code, derived from capital letters in the name. |
| `zone` | String | Railway zone (Northern, Eastern, North Eastern, Western, Central, South Eastern, Southern, East Central). Drives the zone maps/bars. |
| `station_category` | String | A1 / A / B / C — Indian Railways-style classification by footfall/importance. A1 = busiest. Drives the "Performance by Station Category" and heatmap visuals. |
| `state` | String | Indian state the station is in. |
| `latitude` / `longitude` | Float | Coordinates for the map visuals (Passenger Footfall by Zone map, Average Delay by Zone map). |
| `platform_count` | Integer | Number of platforms — context field, can drive tooltips. |
| `is_junction` | Integer (0/1) | 1 if the station is a junction. |

### 2.2 `dim_trains.csv` — Train Master

| Column | Type | Description |
|---|---|---|
| `train_id` | String (PK) | Unique train key, e.g. `TR001`. |
| `train_number` | String | 5-digit train number (cosmetic realism). |
| `train_name` | String | Full train name, e.g. "Delhi Mumbai Rajdhani Express". |
| `train_type` | String | Rajdhani / Vande Bharat / Duronto / Shatabdi / Superfast / Express / Other. This is the field behind "Revenue Contribution by Train Type." |
| `zone` | String | Zone the train's origin station belongs to. |
| `source_station_id` / `destination_station_id` | String (FK) | Links to `dim_stations.station_id`. Defines the fixed route. |
| `distance_km` | Integer | Route distance — used for Avg Journey Distance KPI and fare calc. |
| `avg_speed_kmph` | Float | Average running speed — used to derive journey duration. |
| `fare_per_km` | Float | Base fare rate (₹/km) for that train, before class multiplier. |
| `classes_operated` | String | Pipe-delimited list of classes this train runs (e.g. `1A|2A|3A`). Informational — the actual class-level rows live in the fact table. |
| `weekly_frequency` | Integer | How many days a week the train runs (1–7). |
| `total_coaches` | Integer | Coach count — context field. |

### 2.3 `dim_calendar.csv` — Date Dimension

| Column | Type | Description |
|---|---|---|
| `date` | Date (PK) | One row per calendar day, 2024-01-01 → 2026-12-31. |
| `day_name` | String | Monday…Sunday — drives the delay-by-day heatmap. |
| `day_num` | Integer | Day of month. |
| `week_num` | Integer | ISO-ish week number. |
| `month` / `month_name` | Integer/String | Month number and name. |
| `quarter` | Integer | 1–4. |
| `year` | Integer | Calendar year. |
| `fiscal_year` | String | Indian fiscal year label (`FY25` = Apr 2024–Mar 2025). |
| `is_weekend` | Integer (0/1) | 1 for Sat/Sun. |
| `is_holiday` | Integer (0/1) | 1 on major national holidays. |
| `holiday_name` | String | Name of the holiday, blank otherwise. |
| `season` | String | Regular / Festive Season (Oct–Nov) / Summer Vacation (Apr–Jun) / Winter Holidays (Dec–Jan) / Monsoon (Jul–Sep). Drives seasonality in the data and can drive a filter/parameter. |
| `is_actual` | Integer (0/1) | 1 for dates ≤ 17 Jul 2026 (today). Both fact tables only contain rows where this is 1 — the remaining calendar dates (through Dec 2026) exist purely so a **Forecast** trend line (native Tableau forecasting) has room to draw into, matching the dashed "Forecast" segment in the reference Passenger Trend chart. |

### 2.4 `fact_train_bookings.csv` — Main Fact Table (train × date × class)

| Column | Type | Description |
|---|---|---|
| `booking_id` | String (PK) | Unique row id. |
| `travel_date` | Date (FK → dim_calendar) | The date this train ran. |
| `train_id` | String (FK → dim_trains) | Which train. |
| `source_station_id` / `destination_station_id` | String (FK → dim_stations) | Route endpoints. |
| `class_of_travel` | String | 1A / 2A / 3A / Sleeper / General / Chair Car / Executive Chair Car. |
| `booking_channel` | String | IRCTC Online / Mobile App / PRS Counter / Travel Agent / UTS. |
| `passenger_count` | Integer | **Aggregated** passengers booked in that class, on that train, on that date (this is why 122K rows can represent ~20M+ total passengers — always use `SUM()`). |
| `male_passengers` / `female_passengers` / `other_passengers` | Integer | Gender split of `passenger_count`. Sums to `passenger_count`. |
| `fare_per_passenger` | Float (₹) | Average fare per ticket for that class/train/date. |
| `ticket_revenue` | Float (₹) | `fare_per_passenger × passenger_count`. |
| `refund_amount` | Float (₹) | Refunds issued against that row (0 if none). |
| `net_ticket_revenue` | Float (₹) | `ticket_revenue − refund_amount`. |
| `distance_km` | Integer | Journey distance (repeated from `dim_trains` for convenience — avoids a relationship hop when you just need distance on a booking-level viz). |
| `journey_duration_hrs` | Float | Scheduled + delay-adjusted duration. |
| `delay_minutes` | Float | Delay for that train on that date (same value across all classes of the same train/date — delay is a train-day property, not a class property). |
| `on_time_flag` | Integer (0/1) | 1 if `delay_minutes ≤ 10`. |
| `satisfaction_score` | Float (1–5) | Average passenger satisfaction for that segment. |

### 2.5 `fact_station_daily_ops.csv` — Station Operations Fact (station × date)

| Column | Type | Description |
|---|---|---|
| `ops_id` | String (PK) | Unique row id. |
| `date` | Date (FK → dim_calendar) | Operating date. |
| `station_id` | String (FK → dim_stations) | Which station. |
| `footfall` | Integer | Total people through the station that day (broader than ticketed passengers — includes visitors, senders/receivers, platform ticket holders). |
| `platform_occupancy_pct` | Float | % of platform capacity in use, daily average. |
| `cleanliness_score` | Float (1–10) | Daily cleanliness audit score. |
| `complaints_received` / `complaints_resolved` | Integer | Complaint volume and resolution count that day. |
| `staff_on_duty` / `staff_required` | Integer | Actual vs required staffing — feeds the Staff Utilization gauge (`staff_on_duty / staff_required`). |
| `trains_handled` | Integer | Number of train services that station handled that day. |
| `on_time_trains_count` / `delayed_trains_count` | Integer | Split of `trains_handled`. |
| `avg_delay_minutes` | Float | Station-level average delay that day. |
| `catering_revenue` / `parking_revenue` / `advertising_revenue` / `retail_revenue` / `other_non_ticket_revenue` | Float (₹) | The 5 non-ticket revenue streams — sum these for "Non-Ticket Revenue" and use them individually for the "Top 5 Sources of Non-Ticket Revenue" chart. |

---

## 3. Tableau Setup — Step by Step

1. **New Data Source** → Text File → connect all 5 CSVs into one Data Source ("one data
   source" as you wanted).
2. Drag `fact_train_bookings` onto the canvas first (it's your primary table).
3. Drag `dim_trains` next to it → Tableau will prompt for a relationship → set
   `train_id = train_id`.
4. Drag `dim_stations` next to `fact_train_bookings` → relate on
   `source_station_id = station_id`.
5. Drag `dim_stations` onto the canvas **again** (Tableau lets you add the same table twice)
   → rename it `Destination Station` (double-click the table title) → relate it to
   `fact_train_bookings` on `destination_station_id = station_id`.
6. Drag `dim_calendar` → relate to `fact_train_bookings` on `travel_date = date`.
7. Drag `fact_station_daily_ops` onto a **separate area** of the canvas → relate to
   `dim_stations` (the original instance) on `station_id = station_id`, and to
   `dim_calendar` on `date = date`.
8. Rename the data source to something clean: **"Railway Ops & Passenger Analytics"**.
9. On each dimension table, right-click → set default properties: format `Revenue`/`Amount`
   fields as Currency (₹, no decimals), `*_pct` and `*_score` as Number (2 dp).
10. Go to each sheet and check the little link icon next to field names in the Data pane —
    it tells you which logical table a field is aggregating from. This matters a lot once
    you start mixing station-ops measures with booking measures on the same dashboard (you
    generally won't put them on the *same sheet*, but you will put them on the same
    *dashboard*).

---

## 4. Global Calculated Fields (build these first — used across multiple pages)

### 4.1 Basic building blocks

```
// Total Passengers
SUM([Passenger Count])

// Total Revenue (Ticket + Non-Ticket, Cr)
// Built as a combined measure — see LOD section 4.3 for the non-ticket blend
SUM([Ticket Revenue]) / 10000000

// Net Revenue
SUM([Net Ticket Revenue]) / 10000000

// Avg Delay (mins)
AVG([Delay Minutes])

// On-Time Performance %
SUM([On Time Flag]) / COUNT([Booking Id])
// Note: this counts booking-rows, not distinct train-runs. For a cleaner version at
// train-day grain, use the LOD version in 4.3.

// Avg Satisfaction Score
AVG([Satisfaction Score])
```

### 4.2 Date intelligence (needed for every "▲ x% vs PY" KPI tile)

```
// Selected Period Flag  (paired with a relative-date Parameter, see Section 5)
// Simplify by using Tableau's built-in date filters + a dedicated "vs PY" calc:

// Prior Year Passengers
{ FIXED : SUM(
    IF YEAR([Travel Date]) = YEAR(TODAY())-1 THEN [Passenger Count] END) }

// Cleaner, parameter-driven version (recommended) — see Section 5 for [Comparison Year]
// This Year Passengers
SUM(IF YEAR([Travel Date]) = [Selected Year] THEN [Passenger Count] END)

// Last Year Passengers
SUM(IF YEAR([Travel Date]) = [Selected Year]-1 THEN [Passenger Count] END)

// YoY % Passengers
(SUM(IF YEAR([Travel Date])=[Selected Year] THEN [Passenger Count] END)
 - SUM(IF YEAR([Travel Date])=[Selected Year]-1 THEN [Passenger Count] END))
/ SUM(IF YEAR([Travel Date])=[Selected Year]-1 THEN [Passenger Count] END)
```
Duplicate this pattern for Revenue, Avg Delay, On-Time %, Satisfaction — that's exactly how
every KPI tile in the reference dashboard gets its little "▲ 12.6% vs PY" subtitle.

### 4.3 LOD Expressions (this is where you show you know Tableau)

```
// 1. Train-Day Avg Delay (FIXED) — collapses class-level rows back to one delay value
//    per train-day, so your delay KPIs aren't accidentally weighted by class row-count
{ FIXED [Train Id], [Travel Date] : AVG([Delay Minutes]) }

// 2. On-Time Performance % at the correct grain (train-day, not booking-row)
//    Numerator/denominator both built on the FIXED delay above
{ FIXED [Train Id],[Travel Date] : MAX([On Time Flag]) }   // call this [Train-Day On Time]
// then on the viz:  SUM([Train-Day On Time]) / COUNTD([Train Id]+STR([Travel Date]))

// 3. Revenue per Passenger by Station Category (FIXED, ignores whatever filter context
//    the viz is in — good for a "benchmark line" reference)
{ FIXED [Station Category] : SUM([Ticket Revenue]) / SUM([Passenger Count]) }

// 4. Top Station Flag using INCLUDE — station's footfall rank *within* its own zone,
//    recalculating per row even when the view is aggregated to Zone level
{ INCLUDE [Station Name] : SUM([Footfall]) }

// 5. % of Zone Total using EXCLUDE — lets you show each station's footfall as % of its
//    zone even when Zone is also on the view (classic EXCLUDE use case)
SUM([Footfall]) / { EXCLUDE [Station Name] : SUM([Footfall]) }

// 6. Customer (Passenger) Cohort Growth — FIXED at Train Type + Year, used to power the
//    Revenue Contribution by Train Type donut with year-over-year comparison via a
//    parameter swap
{ FIXED [Train Type], YEAR([Travel Date]) : SUM([Ticket Revenue]) }

// 7. Complaint Resolution Rate (station ops)
SUM([Complaints Resolved]) / SUM([Complaints Received])

// 8. Staff Utilization %
SUM([Staff On Duty]) / SUM([Staff Required])
```

**Why this matters for your write-up:** FIXED = ignore the viz's dimensions entirely and
compute at the level you specify (great for benchmarks/targets). INCLUDE = add a dimension
to the calc's granularity beyond what's on the viz (great for "avg of a detailed metric,
rolled up"). EXCLUDE = remove a dimension the viz has, so the calc runs at a coarser level
(great for %-of-total). Having one real example of each in your workbook — and being able
to explain *why* you picked that one — is exactly what a mentor will probe on.

### 4.4 Table Calculations (for trend/rank visuals)

```
// Running Total Passengers (for cumulative trend if needed)
RUNNING_SUM(SUM([Passenger Count]))

// Rank for "Top 10 Stations by Footfall"
RANK(SUM([Footfall]), 'desc')

// Month-over-Month % change (Monthly Revenue Trend / Monthly Avg Delay Trend)
(ZN(SUM([Ticket Revenue])) - LOOKUP(ZN(SUM([Ticket Revenue])), -1))
/ ABS(LOOKUP(ZN(SUM([Ticket Revenue])), -1))
```

---

## 5. Parameters (build 4 — used as filters across every page)

| Parameter | Type | Values | Used for |
|---|---|---|---|
| `Selected Year` | Integer | 2024, 2025, 2026 (list) | Drives all YoY calcs in Section 4.2. |
| `Date Range Preset` | String | "Last 30 Days", "Last 90 Days", "This Year", "All Time" | Feeds a calculated field that sets a boolean date filter — mimics the "Date Range" dropdown top-right of every reference page. |
| `Delay Threshold (mins)` | Integer | Slider 5–60, default 10 | Lets a mentor interactively redefine "on-time" and watch the On-Time % KPI recompute — great LOD/parameter combo to demo live: `SUM(IF [Delay Minutes] <= [Delay Threshold] THEN 1 ELSE 0 END) / COUNT([Booking Id])`. |
| `Metric Selector` (Executive page) | String | "Passengers", "Revenue", "Avg Delay", "Satisfaction" | Swaps the measure plotted on the Passenger Trend Over Time line chart — classic parameter-swap pattern. Pair with:<br>`CASE [Metric Selector] WHEN "Passengers" THEN SUM([Passenger Count]) WHEN "Revenue" THEN SUM([Ticket Revenue]) WHEN "Avg Delay" THEN AVG([Delay Minutes]) WHEN "Satisfaction" THEN AVG([Satisfaction Score]) END` |

Also build **standard quick filters** (not parameters) for: `Zone`, `Train Type`,
`Class of Travel` — these are dimension filters, shown top-right in the reference as
Date Range / Zone / Train Type / Class dropdowns. Set them as **context filters** if you
notice LOD `FIXED` calcs not responding the way you expect (context filters apply *before*
FIXED LODs; regular filters apply after unless the LOD dimensions are also filtered — this
distinction is worth a sentence in your doc too).

---

## 6. Page-by-Page Build Spec

Global page setup for all 4 dashboards: canvas size **1600 × 1000** (fixed size, matches a
16:10-ish widescreen), left vertical nav rail (Overview / Operations / Revenue / Passengers
/ Reports) built once as a reusable set of navigate-to-sheet buttons + icons, top filter bar
(Date Range, Zone, Train Type, Class), and a bottom-left "Insights" text box with 2 dynamic
sentences built from calculated fields (e.g., `"Passenger traffic grew " + STR(ROUND([YoY % Passengers]*100,1)) + "% compared to last year."`).

Color themes (match the 4 reference pages exactly):
- Page 1 Overview → dark teal (`#0B4F4A`-ish) sidebar, teal/coral KPI accents
- Page 2 Operations → warm orange/cream sidebar and background
- Page 3 Revenue → lavender/purple sidebar and background
- Page 4 Passengers → navy sidebar, blue/pink accents

Build these as 4 separate dashboards in one workbook, then a 5th "Home"/navigation flow if
you want a single entry point (optional polish).

### 6.1 Page 1 — Executive Overview

**KPI tiles (5):**
| Tile | Field | Calc |
|---|---|---|
| Total Passengers | `SUM([Passenger Count])` formatted `8.62M` style (custom number format `0.00,,"M"` after aggregating in millions, or use a calc `SUM([Passenger Count])/1000000`) | subtitle = YoY calc from 4.2 |
| Total Train Journeys | `COUNTD([Train Id] + STR([Travel Date]))` (distinct train-day runs) | subtitle = YoY |
| Total Revenue | `SUM([Ticket Revenue])/10000000` formatted `₹0.0,,"Cr"` | subtitle = YoY |
| Avg Delay (mins) | `AVG([Delay Minutes])` | subtitle = YoY (note: down is *good* here — flip the color logic, red arrow should mean delay *increased*) |
| Avg Satisfaction Score | `AVG([Satisfaction Score])` formatted `0.00" / 5"` | subtitle = YoY |

Build each KPI as its own worksheet: one Text/BAN-style sheet with the big number, a small
sparkline or nothing, and the % change as a second text mark colored via a calculated field
(`IF [YoY %] >= 0 THEN "▲" ELSE "▼" END`, green/red via a diverging color on the axis).

**Passenger Trend Over Time (line chart, dual: Actual solid + Forecast dashed):**
- Columns: `MONTH([Travel Date])` continuous
- Rows: `SUM([Passenger Count])`
- Right-click the axis → **Forecast → Show Forecast**. Tableau will auto-project using
  exponential smoothing into the empty (non-`is_actual`) calendar space through Dec 2026 —
  this is exactly the dashed segment in the reference. Format the forecast line as dashed
  via Format → Forecast.
- If `[Metric Selector]` parameter is built (Section 5), swap `SUM([Passenger Count])` for
  the parameter-driven calc instead, and rename the sheet title dynamically.

**Revenue Contribution by Train Type (colored KPI-tile grid, not a pie):**
- The reference actually renders this as 6 colored blocks with % + ₹ value, not a chart.
  Build it as a **highlight table** or a small-multiples text table: Rows = `Train Type`,
  Text = `SUM([Ticket Revenue])/10000000` and `SUM([Ticket Revenue]) / TOTAL(SUM([Ticket Revenue]))`.
  Color each block by `Train Type` using a custom categorical palette (navy/orange/teal/coral/purple/grey).

**Passenger Footfall by Zone (map):**
- Use `dim_stations.latitude/longitude`, generate a filled map or symbol map of India,
  color/size by `SUM([Footfall])` from `fact_station_daily_ops`, `Zone` on Detail and
  Tooltip. Add a Low→High sequential legend (matches the reference gradient legend).
- Pair with a side table: Rows = `Zone`, Text = `SUM([Footfall])`.

**Booking Channel Share (donut):**
- Dimension: `Booking Channel`, Angle: `SUM([Passenger Count])`.
- Build the donut with two concentric pie marks (outer = data, inner = white circle) — the
  classic Tableau donut trick (dual-axis, one pie sized small and colored white/background).

**Top 10 Stations by Footfall (bar-in-table):**
- Rows: `Station Name` (sorted desc by footfall), Text: `SUM([Footfall])`, plus a small
  in-cell bar via a calculated field driving a Gantt/bar mark, and a `% vs PY` column using
  the pattern from Section 4.2 applied to footfall instead of passengers.
- Use `{FIXED [Station Name] : SUM([Footfall])}` + `RANK()` table calc, filter Rank ≤ 10.

### 6.2 Page 2 — Station Operations Dashboard

**KPI tiles (5):** Avg Delay (mins), On-Time Performance % (LOD calc from 4.3 item 2),
Platform Occupancy % (`AVG([Platform Occupancy Pct])`), Cleanliness Score
(`AVG([Cleanliness Score])`), Complaints Resolved % (LOD 4.3 item 7).

**Average Delay by Station Category & Day (heatmap):**
- Columns: `Day Name` (sort Mon→Sun manually via a calculated sort field
  `CASE [Day Name] WHEN "Monday" THEN 1 ... WHEN "Sunday" THEN 7 END`), Rows:
  `Station Category`, Color: `AVG([Delay Minutes])` on a red-diverging or Orange-sequential
  palette (Low→High delay, matches reference).
- Grain issue to watch: delay lives in `fact_train_bookings` at train-day level, but
  category lives on `dim_stations`. Use the `source_station_id` relationship path.

**Average Delay by Zone (map + ranked bar list):**
- Same India map pattern as Page 1, colored by `AVG([Delay Minutes])`.
- Side list: Rows = `Zone` sorted desc by avg delay, Text = value, small in-cell bar.

**Footfall vs Platform Occupancy (bubble scatter):**
- Columns: `AVG([Platform Occupancy Pct])`, Rows: `AVG([On Time Trains Count]/[Trains Handled])`
  (or reuse On-Time % LOD), Size: `SUM([Footfall])`, Color: `AVG([Cleanliness Score])`
  (diverging), Detail: `Station Category`. This directly mirrors the reference's 4-encoding
  bubble chart (X, Y, Size, Color).

**Performance by Station Category (table):**
- Rows: `Station Category`. Columns: `AVG([Delay Minutes])`, `AVG([Cleanliness Score])`,
  `SUM([Complaints Received])/10000` (per 10K pax — divide by
  `SUM([Footfall])/10000` for a true rate). Add trend arrows via the YoY pattern.

**Staff Utilization (gauge):**
- Build as a **donut gauge**: dual-axis pie, one pie = `SUM([Staff On Duty])/SUM([Staff Required])`
  vs `1 - that`, colored orange/grey; center label = the % via a Text mark layered on top
  (Tableau doesn't have a native gauge — this dual-pie trick is the standard workaround and
  worth explaining in your write-up as a deliberate design choice).

**Top 10 Stations by Avg Delay / Delay Distribution (donut of delay buckets) / Monthly Avg
Delay Trend (line):** same rank/donut/trend patterns as Page 1's equivalents, just delay
instead of footfall. For Delay Distribution, first build a calculated bucket field:
```
CASE
  WHEN [Delay Minutes] <= 5 THEN "On Time (≤5)"
  WHEN [Delay Minutes] <= 15 THEN "5 - 15 mins"
  WHEN [Delay Minutes] <= 30 THEN "15 - 30 mins"
  WHEN [Delay Minutes] <= 60 THEN "30 - 60 mins"
  ELSE "60+ mins"
END
```

### 6.3 Page 3 — Revenue Analytics Dashboard

**KPI tiles (5):** Total Revenue, Ticket Revenue (`SUM([Ticket Revenue])`), Non-Ticket
Revenue (`SUM([Catering Revenue])+SUM([Parking Revenue])+SUM([Advertising Revenue])+SUM([Retail Revenue])+SUM([Other Non Ticket Revenue])`
— build this as one combined calc `[Total Non-Ticket Revenue]`), Refund Amount
(`SUM([Refund Amount])`), Net Revenue (`SUM([Net Ticket Revenue]) + [Total Non-Ticket Revenue]`).

**Revenue Waterfall:**
- Classic Tableau waterfall: a bar chart of `Ticket Revenue`, `+Non-Ticket Revenue`,
  `-Refunds`, `=Net Revenue` using a running-sum Gantt-bar technique:
  1. Build a small manual table (or a calculated field with a `CASE` on a "Category" field
     you construct via a union of the 4 revenue components) — simplest path is a **Union**
     of 4 one-row extracts, or fake it with 4 hardcoded calculated fields + a Gantt chart
     where bar length = value and bar start = `RUNNING_SUM(previous values)`.
  2. Mark type: Gantt Bar. Size = the value. Use `RUNNING_SUM(SUM([Value])) - SUM([Value])`
     as the starting point on the axis.
- This is the single trickiest chart in the whole project — budget extra time and expect to
  look up "Tableau waterfall chart running sum Gantt" once; it's a well-documented pattern.

**Revenue by Zone (bar):** Rows = `Zone`, Columns = `SUM([Ticket Revenue])/10000000`, sorted desc.

**Monthly Revenue Trend (stacked bar, Ticket vs Non-Ticket):**
- Columns: `MONTH([Travel Date])`, Rows: `SUM([Ticket Revenue])` and
  `[Total Non-Ticket Revenue]` stacked via dual measures on Color (Measure Names/Measure
  Values, or two separate Measures pill approach).
- Note grain mismatch: ticket revenue is per-booking, non-ticket is per-station-day. Build
  this as **two separate sheets on a dashboard, overlaid**, or pre-aggregate both to
  Month level as a blended calc — cleanest is two worksheets floated together since they
  come from different fact tables anyway.

**Top 10 Trains by Revenue:** Rows = `Train Name`, sorted desc by `SUM([Ticket Revenue])/10000000`, Rank filter ≤10.

**Revenue Contribution by Passenger Class (donut):** Dimension = `Class of Travel`, Angle = `SUM([Ticket Revenue])`.

**Refund % by Train Type (bar):** Rows = `Train Type`, Columns =
`SUM([Refund Amount]) / SUM([Ticket Revenue])`, sorted desc, formatted as %.

**Top 5 Sources of Non-Ticket Revenue (donut):** Union/pivot the 5 non-ticket columns into
one Dimension "Revenue Source" + one Measure "Value" (Data pane → select the 5 fields →
right-click → **Pivot** — this is a genuinely useful Tableau data-prep skill to show you
know: turning wide columns into long/tidy format for a chart that needs a category
dimension).

### 6.4 Page 4 — Passenger & Punctuality Dashboard

**KPI tiles (5):** Total Passengers, Unique Passengers proxy (`COUNTD([Booking Id])` — note
in the write-up that true unique-rider counts aren't derivable from aggregated booking rows,
so this KPI is *modeled* as distinct bookings; be upfront about this limitation, mentors
respect that), Avg Journey Distance (`AVG([Distance Km])`), Avg Journey Duration
(`AVG([Journey Duration Hrs])`), Punctuality % (On-Time LOD from 4.3).

**Train Punctuality Timeline (Actual vs Forecast line):** same Forecast trick as Page 1,
plotting `[On-Time %]` over `MONTH([Travel Date])`.

**Train Running Timeline (Gantt, sample day):**
- Filter to one specific date (use a parameter `Sample Date`, default = a representative
  weekday).
- Rows: `Train Name`, Mark: Gantt Bar, Path defined by two calculated fields:
  `[Scheduled Start]` (a synthetic time-of-day derived from `HASH([Train Id]) % 24`, or
  simpler — just use `0` as start and `[Journey Duration Hrs]` as bar length) and Size =
  `[Journey Duration Hrs]`.
- This is illustrative in the reference (schedule mock-up), so a simplified same-day Gantt
  using duration only is a perfectly reasonable and honest simplification — note it as a
  design decision in your README/insights text.

**Passenger Gender Distribution (donut):** dimension = pivot of `Male/Female/Other Passengers`
into one "Gender" field (same Pivot technique as 6.3's non-ticket revenue chart) — good,
you reuse the same skill twice, which is worth mentioning as a "reusable technique" in your
documentation.

**Punctuality by Train Type (bar):** Rows = `Train Type`, Columns = On-Time % LOD, sorted desc.

**Delay Distribution (donut):** same bucket calc as 6.2.

**Top 10 Long Distance Routes by Passengers (bar):**
- Build a calculated field `[Route] = [Station Name] + " → " + [Destination Station Name]`
  (using the two `dim_stations` instances from the data model).
- Rows = `Route`, filter `Distance Km` above a threshold (e.g. >800, "long distance"),
  Columns = `SUM([Passenger Count])`, Rank ≤ 10.

**Passenger Satisfaction Score Trend (line with "Latest Score" callout):**
- Columns: `MONTH([Travel Date])`, Rows: `AVG([Satisfaction Score])`.
- Add a reference line at the current value: Format → right-click axis → Add Reference Line
  → Value = `WINDOW_MAX` of the last point → label "Latest Score".

---

## 7. Dashboard Actions (interactivity — another "concepts I want to practice" box ticked)

1. **Filter action**: click a Zone on the Page 1 map → filters the Top 10 Stations table and
   Booking Channel donut on the same dashboard. (Dashboard → Actions → Add Action → Filter,
   Source = map sheet, Target = the other sheets, run on **Select**.)
2. **Highlight action**: hovering a Train Type block on the revenue donut highlights the
   matching bars on Top 10 Trains by Revenue.
3. **Navigate action**: clicking the left-rail icons/buttons switches between the 4
   dashboards (Dashboard → Actions → Go to Sheet, or simpler — use Navigate objects from the
   Dashboard pane, which is built exactly for this and matches the reference's left nav).
4. **Parameter action**: clicking a bar in "Top 10 Stations by Avg Delay" sets the
   `Sample Date`/`Selected Year` parameter used elsewhere, if you want cross-page deep-linking.

---

## 8. Formatting Checklist (the "premium" 10%)

- [ ] Consistent font across all 4 pages (pick one, e.g. "Century Gothic" or default Tableau
      "Tableau Book/Semibold" for titles) — inconsistent fonts are the #1 tell of a rushed
      dashboard.
- [ ] KPI numbers formatted with `M`/`Cr`/`K` suffixes, not raw digits (`8.62M`, not
      `8620000`).
- [ ] Every chart has a clean, sentence-case title — no default "Sheet 12" leftovers.
- [ ] Consistent color mapping for the same dimension across pages — e.g. if "Rajdhani" is
      navy on Page 1's revenue block, keep it navy everywhere it reappears (Format →
      right-click the field in Data pane → Default Properties → Color, so it's global, not
      per-sheet).
- [ ] Tooltips cleaned up (remove default field-list dump; write 1–2 sentence custom
      tooltips with the key number bolded).
- [ ] Zero unnecessary gridlines/borders — reference uses soft card backgrounds with subtle
      drop shadows, no chart-junk gridlines.
- [ ] "Data Last Updated" and "Source" text boxes bottom-right, built from
      `{FIXED : MAX([Travel Date])}` so it auto-updates instead of being hardcoded text —
      small touch, big signal of polish.
- [ ] Insights text box uses dynamic calculated-field-driven sentences (Section 6, page 1
      example) rather than static hardcoded text.
- [ ] Mobile/tablet layout at least sketched (Dashboard → Device Preview) even if you only
      polish Desktop — shows you know the feature exists.

---

## 9. Suggested Build Order (so you don't stall)

1. Connect data source, build relationships (Section 3). Confirm row counts look sane in a
   throwaway sheet (`COUNTD([Booking Id])` should read 122,219).
2. Build all Global Calculated Fields + Parameters (Sections 4–5) before touching any chart.
3. Build Page 1 fully (it reuses the fewest new techniques — good warm-up).
4. Build Page 3 (reuses Page 1's donut/rank/trend patterns, adds the waterfall + pivot).
5. Build Page 2 (adds the heatmap, bubble chart, gauge).
6. Build Page 4 (adds the Gantt, reuses pivot technique from Page 3).
7. Wire up dashboard actions (Section 7) last, once all sheets exist.
8. Formatting pass (Section 8) — do this on a separate day with fresh eyes.
9. Write a 1-paragraph "Approach" README for your submission: what the data represents, why
   a star schema, and 2–3 Tableau concepts you deliberately practiced (LOD, parameters,
   pivot, waterfall) — mentors reward candidates who can articulate *why*, not just *what*.

---

## 10. Known, Honest Simplifications (mention these proactively — it reads as rigor, not weakness)

- `passenger_count` is pre-aggregated per train/date/class, not one row per rider — this is
  what lets 122K rows represent ~20M+ journeys realistically, but it means metrics like
  "unique passengers" are modeled, not literal.
- `delay_minutes` is one value per train-day (applied identically across its class rows) —
  real operations data would have per-coach/per-stop delay, which is out of scope here.
- The Gantt "Train Running Timeline" is illustrative (duration-based), not a true
  minute-by-minute schedule feed.
- Station footfall and train-booking passenger counts are **independently modeled** (footfall
  includes non-ticketed visitors) — don't try to reconcile them to the same total, they're
  intentionally different metrics living in different fact tables.
