# DSA 3050A — Business Intelligence & Data Visualization
## End Semester Practical Examination

**Student Name:** Betelhem Getachew Kebede
**Student ID:** 670549
**Repository:** DSA3050-PowerBI-Betelhem_Kebede-670549
**Software:** Microsoft Power BI Desktop

---

## Project Summary

This project develops a complete Business Intelligence solution analyzing **US Domestic Flight Operations for Q1 2024**. The solution transforms 30,000 raw flight records into an interactive three-page Power BI dashboard that benchmarks airline performance, identifies delay root causes, and supports operational decision-making.

---

## Section A: Dataset Selection & Understanding

### 1. Source of the Dataset

The dataset was obtained from the **United States Bureau of Transportation Statistics (BTS)** — Airline On-Time Performance Data, available at [www.transtats.bts.gov](https://www.transtats.bts.gov). This is a U.S. federal government open-data portal that publishes comprehensive domestic flight operations records under federal reporting requirements — verifiable and authoritative.

The working dataset (`flight_data_2024.csv`) is a stratified random sample of **30,000 records** drawn proportionally across months:

```python
sample = df.groupby("month", group_keys=False).apply(
    lambda x: x.sample(frac=30000/len(df), random_state=42)
)
```

### 2. What the Dataset Represents

Each row represents one US domestic flight operation from January 1 to March 25, 2024, capturing scheduled times, actual times, delay minutes by cause, cancellation status and reason, diversion status, distance, and taxi times.

| Attribute | Value |
|---|---|
| Total Records | 30,000 flight operations |
| Total Columns | 35 variables |
| Date Range | January 1 – March 25, 2024 |
| Carriers Covered | 15 operating airlines |
| Origin Airports | 326 unique airports |
| Destination Airports | 324 unique airports |
| Cancelled Flights | 538 (1.79%) |
| Diverted Flights | 74 (0.25%) |
| Delayed Flights (>15 min) | 5,724 (19.08%) |

### 3. Why This Dataset Was Selected

**Genuine messiness:** Time columns stored as HHMM integers, dates as strings, nulls from cancellations, categorical fields needing standardization — real, not engineered problems.

**Rich analytical potential:** Carrier performance, route analysis, delay cause attribution, temporal trends, and geographic analysis — all examination dimensions simultaneously.

**Real business relevance:** Airline operations analytics is a high-stakes BI domain used by carriers, airports, and regulators for operational and financial decisions.

**Star schema suitability:** Naturally decomposes into a fact table (flights) and dimension tables (carriers, airports, dates).

### 4. Main Variables Available

| Column | Type | Description |
|---|---|---|
| fl_date | Date | Flight date — primary date field |
| op_unique_carrier | Text | IATA airline carrier code |
| origin / dest | Text | Origin and destination airport codes |
| origin_city_name / dest_city_name | Text | Full city names |
| origin_state_nm / dest_state_nm | Text | State names |
| crs_dep_time / dep_time | Integer (HHMM) | Scheduled/actual departure times |
| dep_delay / arr_delay | Float | Departure/arrival delay in minutes |
| cancelled | Integer (0/1) | 1 = cancelled |
| cancellation_code | Text | A=Carrier, B=Weather, C=NAS |
| diverted | Integer (0/1) | 1 = diverted |
| distance | Float | Distance flown in miles |
| carrier_delay | Float | Carrier-caused delay minutes |
| weather_delay | Float | Weather-caused delay minutes |
| nas_delay | Float | NAS delay minutes |
| security_delay | Float | Security delay minutes |
| late_aircraft_delay | Float | Late aircraft delay minutes |
| air_time | Float | Minutes in the air |

### 5. Business / Analytical Problem

**Central question:** How well are US domestic airlines performing operationally in Q1 2024, and which carriers, routes, and time periods present the greatest reliability risks?

Airlines need this continuously because delays and cancellations have direct financial consequences. Understanding which carriers systematically underperform, which delay causes are most prevalent, and which routes carry the greatest risk enables targeted operational improvement.

### 6. Five Analytical Questions

**Q1:** Which airline carriers have the highest on-time performance rates and worst delay rates in Q1 2024?

**Q2:** What are the root causes of flight delays, and which cause — carrier, weather, NAS, security, or late aircraft — is responsible for the most delay minutes?

**Q3:** How do cancellation rates vary by carrier and by cancellation reason (Carrier, Weather, NAS)?

**Q4:** Which routes experience the highest average delays and what is the typical distance profile of delayed flights?

**Q5:** How do delay rates trend across the three months of Q1 2024 — is there a seasonal pattern or a month showing unusual deterioration?

---

## Section B: Power Query — Data Cleaning & Transformation

Nine transformations were performed. All steps are visible in the Applied Steps panel of the `flight_data_2024` query.

### Transformation 1: Correct All Data Types
**Problem:** fl_date stored as text; op_carrier_fl_num as Decimal; several columns auto-detected incorrectly.
**Transformation:** Changed fl_date to Date; numeric columns to Whole Number or Decimal as appropriate.
**Reason:** fl_date must be Date for DateTable relationship and time intelligence DAX functions. Incorrect types affect aggregations and visual formatting.
**Result:** All 35 columns carry correct types. fl_date connects successfully to DateTable.

![Data Types](Screenshots/02_fixed_datatypes..png)

### Transformation 2: Parse Time Columns from HHMM Integer Format
**Problem:** Departure/arrival times stored as HHMM integers (1640 = 16:40). Not usable for time arithmetic — 1710 - 1640 = 70, not 30 minutes.
**Transformation:** Custom columns: `Number.IntegerDivide([crs_dep_time], 100)` for hour, `Number.Mod([crs_dep_time], 100)` for minute.
**Reason:** Proper components enable departure-hour delay analysis and accurate elapsed time calculations.
**Result:** Sched_Dep_Hour column (0–23) available for time-of-day delay analysis.

![Time Parsing](Screenshots/03_parsed_time_columns(custom).png)

### Transformation 3: Create Flight Status and Delay Category Columns
**Problem:** No single column classified each flight's outcome — Cancelled, Diverted, Delayed, On Time required inference from three separate fields.
**Transformation:** Custom column with nested if: Cancelled → Diverted → Delayed (dep_delay > 15) → On Time. Also created Delay Category: Early/On Time, Minor (0–15 min), Moderate (15–60 min), Severe (>60 min).
**Reason:** Single categorical status enables direct status breakdown charts without complex DAX on every visual.
**Result:** Every flight has Flight Status and Delay Category labels powering Page 1 and Page 2 visuals.

![Flight Status](Screenshots/04_status_and_delay_category.png)

### Transformation 4: Conditional Column for Cancellation Reason
**Problem:** cancellation_code contained raw codes (A, B, C) meaningless to business users. Non-cancelled flights showed null.
**Transformation:** Conditional column: A → "Carrier (Airline)", B → "Weather", C → "National Air System", null → "Not Cancelled".
**Reason:** Dashboard visuals must be immediately readable without a code reference. Replaces null with a descriptive label preventing blank entries in visuals.
**Result:** 538 cancellations labelled: 187 Carrier, 326 Weather, 25 NAS.

![Conditional Column](Screenshots/05_conditional_column.png)

### Transformation 5: Merge Queries to Add Carrier Full Name
**Problem:** op_unique_carrier contained two-character codes (AA, WN) unreadable to non-aviation audiences.
**Transformation:** Created DimCarrier table (carrier_code, carrier_name). Used Home → Merge Queries (Left Outer Join) on carrier_code.
**Reason:** Business dashboards should display "Southwest Airlines" not "WN". Left Outer Join retains all 30,000 records.
**Result:** Every flight record has full carrier name. Bar charts show readable airline names.

![Merge Carrier](Screenshots/06_merge_dimcarrier.png)

### Transformation 6: Split Time Columns
**Problem:** HHMM format time values needed splitting into hour components for departure-hour analysis.
**Transformation:** Split Column and custom formulas to extract Sched_Dep_Hour (0–23) as standalone integer.
**Reason:** Departure hour is critical — delays compound through the day as aircraft accumulate lateness on prior legs.
**Result:** Sched_Dep_Hour column enables time-of-day delay pattern analysis.

![Split Columns](Screenshots/07_Splitted_Columns.png)

### Transformation 7: Remove Blank Rows and Errors
**Problem:** After type corrections, some fully blank rows and type conversion errors appeared.
**Transformation:** Home → Remove Rows → Remove Blank Rows and Remove Errors.
**Reason:** Blank rows inflate COUNTROWS() and create blank categories in filters. Errors prevent numeric aggregations.
**Result:** All 30,000 records are clean genuine flight operations.

![Blank Rows](Screenshots/08_clean_blank_rows.png)

### Transformation 8: Create Route Column (Merge Columns)
**Problem:** Route-level analysis required combining origin and destination — no single field existed.
**Transformation:** Merged origin and dest columns with " → " separator creating Route (e.g. "DFW → AMA").
**Reason:** Single Route field enables direct use in charts and tables without DAX workarounds. Arrow separator shows directionality intuitively.
**Result:** Route column available for top delayed routes analysis.

![Route Column](Screenshots/09_route_column(merging%20columns).png)

### Transformation 9: Create Dimension Tables
**Problem:** Flat file repeats carrier names and airport details across thousands of rows — unsuitable for star schema.
**Transformation:** Reference Queries from fact table → select relevant columns → Remove Duplicates → DimCarrier (15), DimOriginAirport (326), DimDestAirport (324).
**Reason:** Star schema requires normalized dimensions for proper one-to-many relationships and efficient filtering.
**Result:** Four queries: flight_data_2024 (fact) + DimCarrier + DimOriginAirport + DimDestAirport.

![Dimension Tables](Screenshots/10_Creating_dimention_tables.png)

---

## Section C: Data Modelling

The model follows a **Star Schema** with `FactFlights` at the centre connected to four dimension tables.

![Data Model](Screenshots/11_Data_Modelling.png)
![Date Table](Screenshots/12_date_Table.png)

### Model Explanation

**FactFlights (flight_data_2024)** is the central fact table — one row per flight operation with all quantitative measures (dep_delay, arr_delay, distance, carrier_delay, weather_delay, nas_delay, security_delay, late_aircraft_delay, cancelled, diverted, air_time) and foreign keys to dimensions.

**DateTable** created in Power Query using List.Dates covering January 1 – March 25, 2024. Contains: Date (PK), Year, Month Number, Month Name, Quarter, Week Number, Day of Week Name. Marked as Date Table to enable TOTALYTD, TOTALMTD, PREVIOUSMONTH. Connected via `DateTable[Date] → FactFlights[fl_date]` (one-to-many, single direction).

**DimCarrier** — 15 carriers with carrier_code (PK) and carrier_name. Created because IATA codes are not business-readable. Connected via `DimCarrier[carrier_code] → FactFlights[op_unique_carrier]` (one-to-many, single direction).

**DimOriginAirport** — 326 origin airports with IATA code (PK), city name, state. Enables geographic filtering of departure operations. Connected via `DimOriginAirport[origin] → FactFlights[origin]` (one-to-many, single direction).

**DimDestAirport** — 324 destination airports. Created separately from DimOriginAirport because origin and destination serve different analytical roles — independent filtering without ambiguous filter paths. Connected via `DimDestAirport[dest] → FactFlights[dest]` (one-to-many, single direction).

### Cardinality and Filter Direction

All relationships: **one-to-many, single cross-filter direction**. One-to-many is correct for star schema — each dimension entity relates to many flights. Single direction prevents circular filter paths and performance issues that bidirectional filtering causes in multi-dimension models.

### Modelling Challenge

The same airport IATA code appears as both origin and destination. Resolved by creating two separate dimension tables rather than a single role-playing dimension, avoiding ambiguous filter paths when both slicers are active simultaneously.

---

## Section D: DAX & Business Calculations

15 DAX measures created across three levels. All defined on the FactFlights table.

![All Measures](Screenshots/28_DAX_Measures.png)

### Level 1 — Core KPI Measures

**Measure 1 — Total Flights**
```dax
Total Flights = COUNTROWS(flight_data_2024)
```
![](Screenshots/13_Measure1_Total_Flights.png)
Counts all records in current filter context. Foundation KPI and denominator for all rate calculations.

**Measure 2 — Delayed Flights**
```dax
Delayed Flights =
CALCULATE(
    COUNTROWS(flight_data_2024),
    flight_data_2024[dep_delay] > 15,
    flight_data_2024[cancelled] = 0
)
```
![](Screenshots/14_Measure2_delayed_Flights.png)
Counts flights delayed >15 minutes (FAA standard) excluding cancellations which have null dep_delay.

**Measure 3 — On Time Flights**
```dax
On Time Flights =
CALCULATE(
    COUNTROWS(flight_data_2024),
    flight_data_2024[dep_delay] <= 15,
    flight_data_2024[cancelled] = 0,
    flight_data_2024[diverted] = 0
)
```
![](Screenshots/15_Measure3_OnTimeFlights.png)

**Measure 4 — Delay Rate**
```dax
Delay Rate = DIVIDE([Delayed Flights], [Total Flights], 0)
```
![](Screenshots/16_Measure4_Delay_Rate.png)
Normalized delay benchmark. DIVIDE handles zero denominator safely. Overall: 19.08%.

**Measure 5 — Cancelled Flights**
```dax
Cancelled Flights =
CALCULATE(COUNTROWS(flight_data_2024), flight_data_2024[cancelled] = 1)
```
![](Screenshots/17_Measure5_CancelledFlights.png)
Total: 538 cancellations (1.79%).

**Measure 6 — Cancellation Rate**
```dax
Cancellation Rate = DIVIDE([Cancelled Flights], [Total Flights], 0)
```
![](Screenshots/18_Measure6_CancellationRate.png)

### Level 2 — Calculated Business Measures

**Measure 7 — Diverted Flights**
```dax
Diverted Flights =
CALCULATE(COUNTROWS(flight_data_2024), flight_data_2024[diverted] = 1)
```
![](Screenshots/19_Measure7_DivertedFlights.png)
Total: 74 diversions (0.25%).

**Measure 8 — Diversion Rate**
```dax
Diversion Rate = DIVIDE([Diverted Flights], [Total Flights], 0)
```
![](Screenshots/20_Measure8_DiversionRate.png)

**Measure 9 — Average Departure Delay**
```dax
Avg Departure Delay = AVERAGE(flight_data_2024[dep_delay])
```
![](Screenshots/21_Measure9_AvgDepartureDelay.png)
Dataset average: 11.24 minutes. AVERAGE excludes nulls (cancelled flights) automatically.

**Measure 10 — Average Arrival Delay**
```dax
Avg Arrival Delay = AVERAGE(flight_data_2024[arr_delay])
```
![](Screenshots/22_Measure10_AVGArrivalDelay.png)
Dataset average: 5.3 minutes — lower than departure delay, confirming crews recover time in flight.

**Measure 11 — Average Flight Distance**
```dax
Avg Flight Distance = AVERAGE(flight_data_2024[distance])
```
![](Screenshots/23_Measure11_AVGFlightDistance.png)
Dataset average: 834.5 miles.

**Measure 12 — Total Delay Minutes**
```dax
Total Delay Minutes =
SUMX(
    flight_data_2024,
    flight_data_2024[carrier_delay] +
    flight_data_2024[weather_delay] +
    flight_data_2024[nas_delay] +
    flight_data_2024[security_delay] +
    flight_data_2024[late_aircraft_delay]
)
```
![](Screenshots/24_Measure12_TotalDelayMinutes.png)
SUMX iterates row by row — handles nulls correctly across five columns. Dataset total: ~406,000–436,000 minutes.

### Level 3 — Advanced DAX

**Advanced Measure 1 — Carrier Delay Rank**
```dax
Carrier Delay Rank =
RANKX(
    ALL(flight_data_2024[op_unique_carrier]),
    [Delay Rate],
    ,
    DESC,
    Dense
)
```
![](Screenshots/25_AdvancedMeasure(CarrierDelayRank).png)
Ranks all 15 carriers by delay rate simultaneously. ALL() removes carrier filter context. Rank 1 = worst.

**Advanced Measure 2 — Delay Performance**
```dax
Delay Performance =
VAR CarrierDelayRate = [Delay Rate]
VAR OverallDelayRate = CALCULATE([Delay Rate], ALL(flight_data_2024))
RETURN
IF(
    CarrierDelayRate <= OverallDelayRate,
    "Above Average",
    "Below Average"
)
```
![](Screenshots/26_AdvancedMeasure(DelayPerformance).png)
VAR stores intermediate values. CALCULATE with ALL() computes overall rate ignoring all filters. IF returns descriptive label.

**Advanced Measure 3 — Delay Rate vs Overall**
```dax
Delay Rate vs Overall =
VAR CarrierRate = [Delay Rate]
VAR OverallRate = CALCULATE([Delay Rate], ALL(flight_data_2024))
RETURN CarrierRate - OverallRate
```
![](Screenshots/27_Advanced_Measure(DelayRate_vs_Overall).png)
Percentage point gap — positive = worse than average, negative = better.

### Six Most Important Measures Explained

| Measure | What | Why Useful | Key Functions | Filter Context | Dashboard Use |
|---|---|---|---|---|---|
| Total Flights | COUNTROWS all records | Primary volume KPI and denominator | COUNTROWS | All slicers | KPI card all pages; rate denominators |
| Delay Rate | Delayed Flights / Total Flights | Normalized benchmark across carriers | DIVIDE | Carrier/month/state slicers | KPI card, carrier bar chart (Page 1), carrier table (Page 3) |
| Total Delay Minutes | SUMX row-by-row sum of 5 delay columns | Absolute impact + cause breakdown | SUMX (iterator) | Carrier and month filters | Bar chart by cause (Page 2), decomposition tree (Page 3) |
| Carrier Delay Rank | RANKX across all 15 carriers | Immediate competitive positioning | RANKX, ALL() | ALL() overrides carrier filter | Carrier performance table (Page 3) |
| Delay Performance | VAR + CALCULATE/ALL + IF label | Contextualizes vs dataset average | VAR, CALCULATE, ALL, IF | ALL() computes global rate independently | Conditional formatting, carrier table (Page 3) |
| Delay Rate vs Overall | Percentage point gap from average | Quantifies gap magnitude | VAR, CALCULATE, ALL | ALL() for global rate | Diverging color bar chart (Page 3) |

---

## Section E: Professional Power BI Dashboards

Three report pages following the narrative: **What happened? → Why did it happen? → Where and who?**

---

### Page 1 — 2024 Flight Operations Executive

![Executive Overview](Screenshots/29_Dashboard_Page1.png)

**Purpose:** High-level executive summary — management understands headline performance within seconds.

**Visuals:**
- 4 KPI Cards: Total Flights, Operational Delay Rate, Severe Delays, Average Delay
- Bar Chart: Operational Delay Rate % by Carrier Name — Envoy Air and JetBlue worst; SkyWest best (~17%)
- Donut Chart: Total Flights by Delay Category — 61.76% on time/early; 5.88% severe delay
- Line Chart: Daily Delay Rate trend across January, February, March 2024
- Slicers: origin_state, dest_city, Code (carrier)

**Story:** The carrier bar chart immediately reveals dramatic performance variation. The donut confirms the majority of flights operate on time. The daily trend shows episodic rather than constant delay problems.

---

### Page 2 — 2024 Flight Delay Diagnostics

![Delay Diagnostics](Screenshots/30_Dashboard_Page2.png)

**Purpose:** Root cause analysis — moving from "what happened" to "why it happened."

**Visuals:**
- 4 KPI Cards: Delayed Flights (11K), Total Delay Minutes (436K), Avg Departure Delay (11.24 min), Delay Rate % (0.36)
- Horizontal Bar Chart: Delay Minutes by Cause — Late Aircraft leads at ~160K, then Carrier (~130K), NAS (~83K), Weather (~30K)
- Pie Chart: Total Flights by Delay Category — 63.93% ahead of schedule/on time
- Multi-Line Chart: All 5 delay causes by month — Late Aircraft consistently dominant; all types dip in February
- Bar Chart: Average Delay by Carrier Code — G4 (Allegiant) ~58 min; AA ~54 min; F9 ~53 min
- Slicers: fl_date, DelayCategory, Code

**Story:** Late Aircraft delay (cascading from earlier legs) is the dominant systemic root cause at 37% of all delay minutes — a structural scheduling problem, not a random event.

---

### Page 3 — Airport and Route Analysis

![Airport and Route Analysis](Screenshots/31_Dashboard_Page3.png)

**Purpose:** Geographic and multi-metric carrier comparison — where delays concentrate and how carriers compare simultaneously.

**Visuals:**
- 4 KPI Cards: Total Late Aircraft Delay (160K), Total Weather Delay (30K), Completed Flights (29K), On-Time/Early Flights (19K)
- Bar/Decomposition Chart: Total Delay Minutes by Code — AA (99,141 min) and WN (74,972 min) = 40% of all delay
- Line Chart: Total Security Delay by month and carrier — AA shows rising trend
- Bubble Map: US geographic distribution — flight volume and delay by origin state
- Carrier Performance Table: Code, Total Flights, Delay Rate %, Avg Delay, Cancelled Flights, Diversions — sortable
- Slicers: origin_city, dest_state

**Story:** The carrier table reveals the key insight — DL (Delta) achieves 29% delay rate with 4,108 flights while AA reaches 43% with only 4,255 flights. A 14-point performance gap at comparable volume proves operational excellence is achievable at scale.

---

## Business Insights

### Insight 1 — Late Aircraft Delay is the Dominant Systemic Root Cause

The Page 2 horizontal bar chart confirms Late Aircraft delay accounts for ~160,000 of ~436,000 total delay minutes (approximately 37%) — the single largest cause, exceeding Carrier Delay (~130K), NAS Delay (~83K), and Weather Delay (~30K). The monthly trend confirms this cause is consistently dominant across all three months.

Late aircraft delay is a cascading effect — delays on early-day flights propagate through all subsequent legs on the same aircraft. This is carrier-controlled and therefore directly actionable.

**Recommendation:** Build greater buffer time into aircraft rotation schedules, particularly at high-congestion hubs. Even a 10–15 minute buffer on the first morning departure of each rotation could substantially reduce cascading delay propagation.

### Insight 2 — A 14-Point Performance Gap Between AA and DL at Comparable Flight Volumes

The Page 3 carrier performance table shows American Airlines (AA) operating 4,255 flights at a 43% delay rate and 54.32 minute average delay, while Delta Air Lines (DL) operates 4,108 flights — a similar scale — at 29% delay rate and 38.83 minute average delay. Carrier Delay Rank and Delay Rate vs Overall confirm this gap.

This 14-point performance difference at comparable volumes proves scale alone does not determine reliability. DL's operational processes achieve materially better performance.

**Recommendation:** AA management should conduct a root-cause comparison against DL — examining aircraft maintenance scheduling, crew management, hub turn-around procedures, and weather response protocols. The Carrier Delay Rank measure enables quarterly monitoring of this gap as a KPI.

### Insight 3 — 35% of Cancellations are Carrier-Controlled and Preventable

Of 538 total cancellations: 326 (60.6%) were Weather-caused (unavoidable), 187 (34.8%) were Carrier-caused (code A — equipment, crew, or operational issues), and 25 (4.6%) were NAS-caused.

While weather-caused cancellations cannot be controlled, the 187 carrier-caused cancellations represent operational failures addressable through investment in maintenance quality, crew reserve planning, and operational resilience.

**Recommendation:** Target the 35% carrier-caused cancellation segment. Even a 50% reduction would prevent ~93 cancellations per quarter — meaningful improvement in passenger reliability and a reduction in compensation costs. Monitor Cancellation Rate and Cancellation Reason monthly using the DateTable time intelligence.

---

## Repository Structure

```
DSA3050-PowerBI-Betelhem_Kebede-670549/
│
├── README.md
│
├── Data/
│   └── flight_data_2024.csv          ← 30,000 row stratified sample from BTS 2024
│
├── Power_BI/
│   └── Power BI.pbix                 ← Full Power BI solution
│
└── Screenshots/
    ├── 01_raw_data.png               ← Section A: Raw dataset
    ├── 02_fixed_datatypes..png       ← Section B: Transformation 1 — data types
    ├── 03_parsed_time_columns(custom).png  ← Section B: Transformation 2 — time parsing
    ├── 04_status_and_delay_category.png    ← Section B: Transformation 3 — status columns
    ├── 05_conditional_column.png     ← Section B: Transformation 4 — cancellation reason
    ├── 06_merge_dimcarrier.png       ← Section B: Transformation 5 — merge queries
    ├── 07_Splitted_Columns.png       ← Section B: Transformation 6 — split columns
    ├── 08_clean_blank_rows.png       ← Section B: Transformation 7 — blank rows
    ├── 09_route_column(merging columns).png ← Section B: Transformation 8 — route column
    ├── 10_Creating_dimention_tables.png    ← Section B/C: Transformation 9 — dimensions
    ├── 11_Data_Modelling.png         ← Section C: Star schema model view
    ├── 12_date_Table.png             ← Section C: DateTable connected
    ├── 13_Measure1_Total_Flights.png       ← Section D: Measure 1
    ├── 14_Measure2_delayed_Flights.png     ← Section D: Measure 2
    ├── 15_Measure3_OnTimeFlights.png       ← Section D: Measure 3
    ├── 16_Measure4_Delay_Rate.png          ← Section D: Measure 4
    ├── 17_Measure5_CancelledFlights.png    ← Section D: Measure 5
    ├── 18_Measure6_CancellationRate.png    ← Section D: Measure 6
    ├── 19_Measure7_DivertedFlights.png     ← Section D: Measure 7
    ├── 20_Measure8_DiversionRate.png       ← Section D: Measure 8
    ├── 21_Measure9_AvgDepartureDelay.png   ← Section D: Measure 9
    ├── 22_Measure10_AVGArrivalDelay.png    ← Section D: Measure 10
    ├── 23_Measure11_AVGFlightDistance.png  ← Section D: Measure 11
    ├── 24_Measure12_TotalDelayMinutes.png  ← Section D: Measure 12
    ├── 25_AdvancedMeasure(CarrierDelayRank).png  ← Section D: Advanced 1
    ├── 26_AdvancedMeasure(DelayPerformance).png  ← Section D: Advanced 2
    ├── 27_Advanced_Measure(DelayRate_vs_Overall).png ← Section D: Advanced 3
    ├── 28_DAX_Measures.png           ← Section D: All measures in Fields pane
    ├── 29_Dashboard_Page1.png        ← Section E: Executive Overview
    ├── 30_Dashboard_Page2.png        ← Section E: Delay Diagnostics
    └── 31_Dashboard_Page3.png        ← Section E: Airport & Route Analysis
```

---

## Data Source Citation

Bureau of Transportation Statistics (BTS). *Airline On-Time Performance Data*. United States Department of Transportation. Available at: https://www.transtats.bts.gov
