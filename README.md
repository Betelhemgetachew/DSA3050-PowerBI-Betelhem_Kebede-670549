# DSA 3050A - Business Intelligence & Data Visualization
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

The dataset was obtained from the **United States Bureau of Transportation Statistics (BTS)** — Airline On-Time Performance Data, available at [www.transtats.bts.gov](https://www.transtats.bts.gov). This U.S. federal government open-data portal publishes comprehensive domestic flight operations records under federal reporting requirements, making it a verifiable and authoritative source.

The working dataset (`flight_data_2024.csv`) is a stratified random sample of **30,000 records** drawn proportionally across months:

```python
sample = df.groupby("month", group_keys=False).apply(
    lambda x: x.sample(frac=30000/len(df), random_state=42)
)
```

### 2. What the Dataset Represents

The dataset represents **US domestic commercial flight operations** for the period **January 1, 2024 to March 25, 2024 (Q1 2024)**. Each row corresponds to one flight segment—one aircraft departing from one airport and arriving at another—and captures the full operational lifecycle including scheduled times, actual times, delay minutes broken down by cause, cancellation status and reason, diversion status, distance flown, and taxi times.

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

**Genuine complexity and messiness.** The raw file contains mixed data types (time columns as HHMM integers, dates as strings), null values across multiple columns due to cancellations, and categorical fields requiring standardization - making it genuinely suitable for substantial Power Query transformation work.

**Rich analytical potential.** The dataset supports simultaneous analysis across multiple dimensions: carrier performance, route analysis, delay cause attribution, temporal trends, and geographic operations - satisfying the examination's complexity requirements.

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

**Q2: What are the root causes of flight delays, and which cause - carrier, weather, NAS, security, or late aircraft - is responsible for the most delay minutes?** Understanding whether delays are carrier-controlled (improvable) or external (weather, NAS) fundamentally changes the recommended operational response.

**Q3:** How do cancellation rates vary by carrier and by cancellation reason (Carrier, Weather, NAS)?

**Q4:** Which routes experience the highest average delays and what is the typical distance profile of delayed flights?

**Q5: How do delay rates and cancellation rates trend across the three months of Q1 2024 - is there a seasonal pattern or a month showing unusual performance deterioration?** Q1 spans the US winter weather season. Identifying whether January shows worse performance than March helps validate weather as a systematic driver and informs seasonal staffing decisions.

---

## Section B: Power Query — Data Cleaning & Transformation

Nine transformations were performed. All steps are visible in the Applied Steps panel of the `flight_data_2024` query.

### Transformation 1: Correct All Data Types
**Problem:** fl_date was stored as text; op_carrier_fl_num as Decimal; several columns auto-detected incorrectly on import.
**Transformation:** Changed fl_date to Date type; op_carrier_fl_num to Whole Number; confirmed cancelled and diverted as Whole Number (0/1); and corrected other numeric columns to Whole Number or Decimal as appropriate.
**Reason:** Correct types are foundational - fl_date must be Date for the DateTable relationship and time intelligence DAX functions to work.
**Result:** All 35 columns carry correct types. fl_date connects successfully to DateTable.

![Data Types](Screenshots/02_fixed_datatypes..png)

### Transformation 2: Parse Time Columns from HHMM Integer Format
**Problem:** crs_dep_time, dep_time, crs_arr_time, arr_time stored as raw integers (e.g. 1640 = 16:40) - not usable for time arithmetic or time-of-day analysis.
**Transformation:** Created custom columns extracting scheduled departure hour using `Number.IntegerDivide([crs_dep_time], 100)` and minute component using `Number.Mod([crs_dep_time], 100)`. Example: extracting hour/minute components prevents incorrect HHMM subtraction (1710 - 1640 = 70).
**Reason:** Proper parsing enables time-of-day grouping and accurate elapsed time calculations.
**Result:** Departure hour available as a standalone column for time-of-day analysis.

![Time Parsing](Screenshots/03_parsed_time_columns(custom).png)

### Transformation 3: Create Flight Status and Delay Category Columns
**Problem:** No single column classified each flight's overall outcome - cancelled, diverted, delayed, or on time had to be inferred from three separate fields.
**Transformation:** Custom column with nested if logic: Cancelled → Diverted → Delayed (dep_delay > 15) → On Time. Also created Delay Category (Early/On Time, Minor, Moderate, Severe).
**Reason:** A single categorical status field enables direct status breakdown charts without complex DAX filters.
**Result:** Every flight has a Flight Status and Delay Category label powering status distribution visuals.

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
**Problem:** Time values in HHMM format needed splitting into hour/minute components for departure-hour analysis.
**Transformation:** Applied Split Column and custom formulas to extract Sched_Dep_Hour (0–23) as a standalone integer column.
**Reason:** Departure hour is critical for delay analysis - delays compound through the day as aircraft accumulate lateness on prior legs.
**Result:** Sched_Dep_Hour column enables time-of-day delay pattern analysis in the Diagnostic page.

![Split Columns](Screenshots/07_Splitted_Columns.png)

### Transformation 7: Remove Blank Rows and Errors
**Problem:** After type corrections, some fully blank rows and type conversion errors appeared.
**Transformation:** Home → Remove Rows → Remove Blank Rows and Remove Errors.
**Reason:** Blank rows inflate COUNTROWS() and create blank categories in filters. Errors prevent numeric aggregations.
**Result:** All 30,000 records are clean genuine flight operations.

![Blank Rows](Screenshots/08_clean_blank_rows.png)

### Transformation 8: Create Route Column (Merge Columns)
**Problem:** Route-level analysis required combining origin and destination - no single field identified the full route.
**Transformation:** Merged origin and dest columns with " → " separator to create Route column (e.g. "DFW → AMA").
**Reason:** A single Route field enables direct use in bar charts and tables for top-delayed-routes analysis without DAX workarounds.
**Result:** Route column powers the top delayed routes table on the Detailed Analysis page.

![Route Column](Screenshots/09_route_column(merging%20columns).png)

### Transformation 9: Create Dimension Tables
**Problem:** Flat file structure repeats carrier names and airport details across thousands of rows - unsuitable for star schema modelling.
**Transformation:** Created DimCarrier, DimOriginAirport, and DimDestAirport using Reference Query from the fact table, selecting relevant columns and applying Remove Duplicates. Resulting objects: DimCarrier (15), DimOriginAirport (326), DimDestAirport (324).
**Reason:** Star schema requires normalized dimensions for proper one-to-many relationships, efficient filtering, and clean model structure.
**Result:** Model has 4 queries: 1 fact (flight_data_2024) + 3 dimensions + DateTable.

![Dimension Tables](Screenshots/10_Creating_dimention_tables.png)

---

## Section C: Data Modelling

The model follows a **Star Schema** with `FactFlights` at the centre connected to four dimension tables.

![Data Model](Screenshots/11_Data_Modelling.png)
![Date Table](Screenshots/12_date_Table.png)

### Model Explanation

**FactFlights (flight_data_2024)** is the central fact table. It contains one row per flight operation and stores all quantitative measures: dep_delay, arr_delay, distance, carrier_delay, weather_delay, nas_delay, security_delay, late_aircraft_delay, cancelled, diverted, air_time, and actual_elapsed_time. These measures are what all DAX calculations aggregate. The fact table was selected because each row represents one transactional event (a single flight) - the core analytical unit of this dataset.

**DateTable** created in Power Query using List.Dates covering January 1 – March 25, 2024. Contains: Date (PK), Year, Month Number, Month Name, Quarter, Week Number, Day of Week Name. Marked as Date Table to enable TOTALYTD, TOTALMTD, PREVIOUSMONTH. Connected via `DateTable[Date] → FactFlights[fl_date]` (one-to-many, single direction).

**DimCarrier** — 15 carriers with carrier_code (PK) and carrier_name. Created because IATA codes are not business-readable. Connected via `DimCarrier[carrier_code] → FactFlights[op_unique_carrier]` (one-to-many, single direction).

**DimOriginAirport** — 326 origin airports with IATA code (PK), city name, state. Enables geographic filtering of departure operations. Connected via `DimOriginAirport[origin] → FactFlights[origin]` (one-to-many, single direction).

**DimOriginAirport** stores 326 unique origin airports with IATA code, city name, and state. Enables geographic filtering of departure operations. Connected via DimOriginAirport[origin] → FactFlights[origin] (one-to-many, single filter direction).

**DimDestAirport** stores 324 unique destination airports. Created separately from DimOriginAirport because origin and destination serve different analytical roles - independent filtering on origin vs destination without ambiguity. Connected via DimDestAirport[dest] → FactFlights[dest] (one-to-many, single filter direction).

### Cardinality and Filter Direction

All relationships use **one-to-many cardinality** - each dimension entity relates to many flight records, which is the correct star schema cardinality. **Single cross-filter direction** was chosen throughout to ensure filters flow from dimensions into the fact table only, preventing circular filter paths and performance degradation that bidirectional filtering can cause in multi-dimension models.

### Modelling Challenge

The same airport IATA code appears as both origin and destination. Resolved by creating two separate dimension tables rather than a single role-playing dimension, avoiding ambiguous filter paths when both slicers are active simultaneously.

---

## Section D: DAX & Business Calculations

15 DAX measures created across three levels. All defined on the FactFlights table.

![All Measures](Screenshots/28_DAX_Measures.png)

### Level 1 - Core KPI Measures

**Measure 1 - Total Flights**
```dax
Total Flights = COUNTROWS(flight_data_2024)
```
![](Screenshots/13_Measure1_Total_Flights.png)
Counts all records in current filter context. Foundation KPI and denominator for all rate calculations.

**Measure 2 - Delayed Flights**
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

**Measure 3 - On Time Flights**
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

**Measure 4 - Delay Rate**
```dax
Delay Rate = DIVIDE([Delayed Flights], [Total Flights], 0)
```
![](Screenshots/16_Measure4_Delay_Rate.png)
Normalized delay benchmark. DIVIDE handles zero denominator safely. Overall: 19.08%.

**Measure 5 - Cancelled Flights**
```dax
Cancelled Flights =
CALCULATE(COUNTROWS(flight_data_2024), flight_data_2024[cancelled] = 1)
```
![](Screenshots/17_Measure5_CancelledFlights.png)
Total: 538 cancellations (1.79%).

**Measure 6 - Cancellation Rate**
```dax
Cancellation Rate = DIVIDE([Cancelled Flights], [Total Flights], 0)
```
![](Screenshots/18_Measure6_CancellationRate.png)

### Level 2 - Calculated Business Measures

**Measure 7 - Diverted Flights**
```dax
Diverted Flights =
CALCULATE(COUNTROWS(flight_data_2024), flight_data_2024[diverted] = 1)
```
![](Screenshots/19_Measure7_DivertedFlights.png)
Total: 74 diversions (0.25%).

**Measure 8 - Diversion Rate**
```dax
Diversion Rate = DIVIDE([Diverted Flights], [Total Flights], 0)
```
![](Screenshots/20_Measure8_DiversionRate.png)

**Measure 9 - Average Departure Delay**
```dax
Avg Departure Delay = AVERAGE(flight_data_2024[dep_delay])
```
![](Screenshots/21_Measure9_AvgDepartureDelay.png)
Dataset average: 11.24 minutes. AVERAGE excludes nulls (cancelled flights) automatically.

**Measure 10 - Average Arrival Delay**
```dax
Avg Arrival Delay = AVERAGE(flight_data_2024[arr_delay])
```
![Avg Arrival Delay](Screenshots/22_Measure10_AVGArrivalDelay.png)
Dataset average: 5.3 minutes - lower than departure delay, confirming crews recover time in flight.

**Measure 11 - Average Flight Distance**
```dax
Avg Flight Distance = AVERAGE(flight_data_2024[distance])
```
![](Screenshots/23_Measure11_AVGFlightDistance.png)
Dataset average: 834.5 miles.

**Measure 12 - Total Delay Minutes**
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

### Level 3 - Advanced DAX

**Advanced Measure 1 - Carrier Delay Rank**
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

**1. Total Flights**
What it calculates: Total count of flight records in the current filter context. Why useful: Foundation KPI — used as denominator in all rate calculations and as the primary volume metric. DAX functions: COUNTROWS. Filter context: Responds to all slicers (carrier, month, route). Dashboard use: KPI card on Executive Overview.

**2. Delay Rate**
What it calculates: Proportion of all flights delayed more than 15 minutes. Why useful: Primary performance benchmark - comparable across carriers, months, and routes on a normalized basis. DAX functions: DIVIDE (safely handles zero denominator), references Delayed Flights and Total Flights measures. Filter context: When carrier slicer active, automatically shows carrier-specific delay rate. Dashboard use: KPI card, carrier comparison table, monthly trend line chart.

**3. Total Delay Minutes (SUMX)**
What it calculates: Aggregate delay minutes across all five delay cause columns for all flights in context. Why useful: Shows the total operational impact of delays in absolute terms and enables delay cause breakdown analysis. DAX functions: SUMX (row-by-row iteration handles nulls correctly across five columns). Filter context: Responds to all filters. Dashboard use: Delay cause breakdown treemap and stacked bar chart on Diagnostic page.

**4. Carrier Delay Rank (RANKX)**
What it calculates: Each carrier's rank from worst (1) to best by delay rate. Why useful: Immediate competitive ranking - identifies worst performing carrier at a glance. DAX functions: RANKX with ALL() to override carrier filter context, Dense ranking. Filter context: ALL() removes carrier filter so all 15 carriers rank simultaneously. Dashboard use: Carrier performance table on Detailed Analysis page.

**5. Delay Performance (VAR + CALCULATE/ALL)**
What it calculates: Whether the current carrier's delay rate is Above Average or Below Average compared to the overall dataset. Why useful: Contextualizes performance - a carrier with 20% delay rate looks bad, but if the industry average is 22% it is actually performing well. DAX functions: VAR for intermediate storage, CALCULATE with ALL() for overall rate, IF() for label output. Filter context: CarrierDelayRate reads current filter context; OverallDelayRate explicitly removes all filters with ALL(). Dashboard use: Conditional formatting on carrier table and Diagnostic Analysis page.

**6. Delay Rate vs Overall (VAR + CALCULATE/ALL)**
What it calculates: Percentage point difference between a carrier's delay rate and the overall average (positive = worse, negative = better). Why useful: Quantifies the magnitude of the performance gap, not just direction. DAX functions: VAR, CALCULATE with ALL(). Filter context: Same as Delay Performance - uses ALL() to compute the overall benchmark independently. Dashboard use: Diverging color bar chart on Detailed Analysis page.

---

## Section E: Professional Power BI Dashboards

Three report pages following the narrative: **What happened? → Why did it happen? → Where and who?**

### Page 1 - Executive Overview
Provides management with immediate understanding of Q1 2024 flight performance.

---

### Page 1 — 2024 Flight Operations Executive

### Page 2 - Carrier & Route Detailed Analysis
Enables operations managers to benchmark carriers and identify problem routes.

### Page 3 - Diagnostic & Root Cause Analysis
Investigates why delays occur, moving from descriptive to diagnostic analytics.

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

**Insight 1 - Late Aircraft Delay is the Dominant Root Cause**
Late aircraft delay accounts for the largest share of aggregate delay minutes and represents a cascading operational problem: delays on early flights propagate into later flights across the same aircraft rotations. Airlines can reduce this by building more buffer time into schedules and improving aircraft turn-around efficiency.

**Recommendation:** Build greater buffer time into aircraft rotation schedules, particularly at high-congestion hubs. Even a 10–15 minute buffer on the first morning departure of each rotation could substantially reduce cascading delay propagation.

**Insight 2 - Carrier Performance Varies Significantly**
Delay rate varies substantially across the 15 carriers. The Carrier Delay Rank measure identifies which carriers systematically underperform relative to the dataset average. Carriers ranked in the bottom third should examine their carrier-caused delay contribution as a primary improvement target.

**Insight 3 - Most Cancellations are Weather-Driven but Carrier Cancellations are Preventable**
Of 538 cancellations: 326 (60.6%) were weather-caused and operationally unavoidable. However, 187 (34.8%) were carrier-caused — these represent operational failures that may be addressable through better maintenance scheduling, crew management, and aircraft reliability programs.

**Recommendation:** Target the carrier-caused cancellation segment. Even a partial reduction would meaningfully improve passenger reliability and reduce compensation costs. Monitor Cancellation Rate and Cancellation Reason monthly using the DateTable time intelligence.

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

Bureau of Transportation Statistics (BTS). *Airline On-Time Performance Data*. United States Department of Transportation. Available at: https://www.transtats.bts.gov/Tables.asp?QO_VQ=EFD&QO_anzr=Nv4yv0r%FDb0-gvzr%FDcr4s14zn0pr%FDQn6n&QO_fu146_anzr=b0-gvzr

---

## Dashboard Screenshots

### Page 1 — 2024 Flight Operations Executive

![Executive Overview](Screenshots/29_Dashboard_Page1.png)

**Page purpose:** High-level executive summary of Q1 2024 flight operations performance.

**Key visuals:**
- 4 KPI cards: Total Flights, Operational Delay Rate, Severe Delays, Average Delay
- Bar chart: Operational Delay Rate % by Carrier Name - ranks all 15 carriers from worst (Envoy Air, JetBlue at ~100% when filtered) to best (SkyWest at ~17%)
- Donut chart: Total Flights by Delay Category - shows 61.76% on time/early, 14.71% minor delay, 5.88% severe
- Line chart: Operational Delay Rate % by fl_date - daily volatility across January, February, March 2024
- Slicers: origin_state, dest_city, Code (carrier)

**Story:** A manager sees in seconds which carriers are underperforming and how delay rates fluctuate day to day across Q1.

---

### Page 2 - 2024 Flight Delay Diagnostics

![Delay Diagnostics](Screenshots/30_Dashboard_Page2.png)

**Page purpose:** Root cause analysis — moving from "what happened" to "why it happened."

**Key visuals:**
- 4 KPI cards: Delayed Flights (11K), Total Delay Minutes (436K), Average Departure Delay (11.24 min), Delay Rate % (0.36)
- Horizontal bar chart: Delay Minutes by Cause - Late Aircraft Delay dominates at ~160K minutes, followed by Carrier Delay (~130K), NAS Delay (~83K), Weather Delay (~30K)
- Pie chart: Total Flights by Delay Category - 63.93% ahead of schedule/on time
- Multi-line chart: Delay cause breakdown by month - all delay types dip in February then rise slightly in March
- Bar chart: Average Delay by Carrier Code - G4 (Allegiant) leads at ~58 min, followed by AA (~54 min) and F9 (~53 min)
- Slicers: fl_date, DelayCategory, Code

**Story:** Late aircraft delay (cascading from earlier legs) is the single largest cause at ~37% of total delay minutes - a systemic scheduling problem, not a random event.

---

### Page 3 - Airport and Route Analysis

![Airport and Route Analysis](Screenshots/31_Dashboard_Page3.png)

**Page purpose:** Geographic and route-level analysis - where delays are concentrated and how carriers compare on all key metrics simultaneously.

**Key visuals:**
- 4 KPI cards: Total Late Aircraft Delay (160K), Total Weather Delay (30K), Completed Flights (29K), On-Time/Early Flights (19K)
- Decomposition bar: Total Delay Minutes by carrier code - AA (99,141 min) and WN (74,972 min) contribute ~40% of all delay
- Line chart: Total Security Delay by month and carrier - American Airlines shows highest and rising security delay trend
- Bubble map: US geographic distribution of flights by origin state - bubble size = flight volume, positioned across all US states
- Carrier performance table: Code, Total Flights, Operational Delay Rate %, Average Delay, Cancelled Flights, Diversions - sortable multi-metric comparison showing AA (43% delay rate, 4,255 flights) vs DL (29% delay rate, 4,108 flights)
- Slicers: origin_city, dest_state

**Story:** Geographic concentration is clear - highest volumes from eastern seaboard states. The performance table reveals DL (Delta) outperforms AA by 14 percentage points despite operating a similar number of flights, proving high volume does not require high delay rates.

---

## Six Most Important DAX Measures - Explained

### 1. Total Flights
**What:** Counts all flight records in the current filter context using `COUNTROWS`.
**Why useful:** Foundation KPI and denominator for all rate calculations.
**Filter context:** Responds to all slicers - carrier, origin, date, delay category.
**Dashboard use:** Primary KPI card on Page 1; denominator in Delay Rate, Cancellation Rate, Diversion Rate.

### 2. Delay Rate
**What:** `DIVIDE([Delayed Flights], [Total Flights], 0)` - proportion of flights delayed >15 min.
**Why useful:** Normalized benchmark enabling fair comparison across carriers of different sizes.
**Filter context:** When carrier slicer active, automatically produces carrier-specific rate.
**Dashboard use:** KPI card, carrier bar chart (Page 1), carrier performance table (Page 3).

### 3. Total Delay Minutes (SUMX)
**What:** `SUMX` iterates row by row summing all five delay cause columns before aggregating.
**Why useful:** Shows absolute impact of delays and enables cause breakdown - SUMX handles nulls correctly across five columns simultaneously.
**Filter context:** Responds to all filters; carrier-specific when Code slicer active.
**Dashboard use:** KPI card (Page 2), horizontal bar chart by delay cause (Page 2), decomposition tree (Page 3).

### 4. Carrier Delay Rank (RANKX)
**What:** `RANKX(ALL(...), [Delay Rate], , DESC, Dense)` — ranks all 15 carriers by delay rate simultaneously.
**Why useful:** Immediate competitive positioning - identifies worst performing carrier at a glance.
**Filter context:** `ALL()` removes carrier filter so all carriers rank against each other even when one is selected.
**Dashboard use:** Carrier performance table (Page 3).

### 5. Delay Performance (VAR + CALCULATE/ALL)
**What:** Uses VAR to store carrier delay rate and overall rate, IF to return "Above Average" or "Below Average".
**Why useful:** Contextualizes performance — a 20% delay rate means different things depending on whether the industry average is 15% or 25%.
**Filter context:** `CALCULATE([Delay Rate], ALL(flight_data_2024))` explicitly removes all filters to compute the true overall rate.
**Dashboard use:** Conditional formatting column in carrier table (Page 3).

### 6. Delay Rate vs Overall (VAR + CALCULATE/ALL)
**What:** Calculates percentage point gap between carrier rate and overall average (positive = worse, negative = better).
**Why useful:** Quantifies the magnitude of the performance gap, not just its direction.
**Filter context:** Same ALL() pattern as Delay Performance — overall rate computed independently of current filter.
**Dashboard use:** Diverging color bar chart on Page 3 (red = worse than average, green = better).
