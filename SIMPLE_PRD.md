---
title: "Clutch Riding Analysis & Data Pipeline"
subtitle: "Final Technical Product Requirements Document"
author: "Data Science Team"
date: "February 5, 2026"
version: "3.3"
classification: "Internal Use"
---

<div style="page-break-after: always;"></div>

# 1. Document Overview

## 1.1 Purpose
This document provides a comprehensive technical specification for the automated data pipeline that analyzes vehicle telemetry data to detect clutch riding events, calculates their impact on fuel efficiency, and generates structured, queryable data tables with robust quality flagging.

## 1.2 Scope
This document covers the end-to-end data pipeline, including data sources, detailed processing logic, data quality checks, the schema of the final output tables, and the design for the customer-facing report.

# 2. Business Objective

## 2.1 Problem Statement
Inefficient driver behavior, specifically "clutch riding," leads to measurable fuel waste and increased vehicle maintenance costs. A systematic, data-driven approach is required to identify, quantify, and flag this behavior at scale across the fleet to enable targeted interventions and cost-saving initiatives.

## 2.2 Solution
An automated pipeline, executed via a Jupyter Notebook (`clutch_riding_production_7days.ipynb`), that transforms raw telemetry into two structured, persistent database tables that directly power a multi-level customer-facing report.

## 2.3 Success Metrics
- **Data Robustness:** The pipeline can successfully process any specified date range of historical data.
- **Hypothesis Validation:** The final dataset clearly demonstrates a statistically significant negative correlation between clutch riding and fuel efficiency.
- **Actionable Insight Generation:** The data proves that for "Long" and "Very Long" duration clutch riding events, mileage is worse than normal riding mileage in over 80% of observed cases.

<div style="page-break-after: always;"></div>

# 3. System Overview

## 3.1 High-Level Architecture
The system follows a two-stage database query process, followed by parallelized in-memory processing, and finally persists the results back to the database.

```mermaid
graph TD
    subgraph "Data Sources"
        A[OS_DB: engineoncycles] --> B;
        C[tracking_db: obd_data_history];
    end

    subgraph "Processing Pipeline (Jupyter)"
        B(Get Cycles) --> E{For each cycle};
        E --> F[Fetch OBD Data];
        F --> G[process_cycle_data];
        G --> H[process_single_day];
    end
    
    subgraph "Final Output"
        H --> I[DB Table: clutch_riding_events];
        H --> J[DB Table: clutch_riding_daily_summary];
    end
```

# 4. Data Sources

## 4.1 Input 1: Engine Cycles
- **Database:** `OS_DB`
- **Table:** `public.engineoncycles`
- **Purpose:** Provides the list of trips (engine on-to-off cycles) to be analyzed.

## 4.2 Input 2: OBD History
- **Database:** `tracking_db`
- **Table:** `public.obd_data_history`
- **Purpose:** Provides the core time-series telemetry for each cycle.

<div style="page-break-after: always;"></div>

# 5. Processing Logic

The core logic is executed by two main functions: `process_cycle_data` and `process_single_day`.

## 5.1 `process_cycle_data` Flow
This function processes the raw dataframe for a single cycle into a structured set of events and quality flags.

```mermaid
graph TD
    subgraph "Input: Raw Cycle DataFrame"
        A[Raw OBD Data];
    end

    subgraph "Processing Steps"
        A --> B(Clean & Typecast Data);
        B --> C(Calculate Interval Features);
        C --> D(Detect Cycle-Level DQ Issues);
        D --> E(Define 'is_clutch_riding' State);
        E --> F(Create Event Groups);
        F --> G(Aggregate each group into an event row);
        G --> H(Calculate Mileage with Fallback);
        H --> I(Calculate Event-Level DQ Flag);
    end

    subgraph "Output"
        I --> J{Dictionary of 'all_events' DataFrame, 'data_quality' dict};
    end
```

### 5.1.1 Event Grouping Logic
Events are created when the clutch riding status changes.

```python
# In process_cycle_data:
df['event_group'] = (df['is_clutch_riding'] != df['is_clutch_riding'].shift()).cumsum()
```

### 5.1.2 Fuel and Mileage Calculation with Fallback
To maximize data reliability, a fallback system is used to determine the definitive mileage for each event.

1.  **Primary Method (from `fuel_rate`):** The system first calculates mileage using the instantaneous `fuel_rate`.
2.  **Validation:** The result is validated against rules (e.g., mileage > 15 km/L).
3.  **Fallback Method (from `fuel_consumption`):** If the primary calculation is invalid, the system uses the mileage calculated from the cumulative `fuel_consumption` column.
4.  **Final Value:** The final `event_mileage_kmpl` column contains the best valid result. Both raw fuel calculations (`event_fuel_from_rate_liters` and `event_fuel_from_consumption_liters`) are kept for transparency.

## 5.2 Data Filtering and Validation Rules
The pipeline generates flags to identify questionable data without deleting it.

### 5.2.1 Event-Level Flags
-   `is_invalid_mileage_flag`: A `BOOLEAN` flag set to `True` if the final mileage for an event is considered invalid (e.g., > 15 km/L, or zero fuel for a moving event).

### 5.2.2 Cycle-Level Flags
-   `has_odometer_reset_flag`: A `BOOLEAN` flag set to `True` if the vehicle's odometer reading went backward during the cycle.
-   `has_data_gap_flag`: A `BOOLEAN` flag set to `True` if the time between any two consecutive data packets exceeded 45 seconds.
-   `is_short_duration_cycle`: A `BOOLEAN` flag set to `True` if the total cycle duration is less than 60 seconds.
-   `is_short_distance_cycle`: A `BOOLEAN` flag set to `True` if the total cycle distance is less than 0.1 km.

<div style="page-break-after: always;"></div>

# 6. OUTPUT SPECIFICATION

The pipeline produces two primary, persistent tables in the database.

## 6.1 Output 1: Event-Level Table
**Proposed Table Name:** `clutch_riding_events`
**Description:** Contains one row for every detected riding event, designed for deep analysis.

| Column Name | Data Type | Description |
|---|---|---|
| `cycle_id` | VARCHAR | Foreign key to the engine cycle |
| `uniqueid` | VARCHAR | Vehicle identifier |
| `event_type` | VARCHAR | 'clutch_riding' or 'normal_riding' |
| `event_start_ts`| INTEGER | Start timestamp of the event |
| `event_end_ts` | INTEGER | End timestamp of the event |
| `event_duration_sec`| FLOAT | Duration of the event in seconds |
| `event_distance_m`| FLOAT | Distance covered during the event in meters |
| `event_fuel_from_rate_liters` | FLOAT | Fuel consumed (from `fuel_rate`) |
| `event_fuel_from_consumption_liters` | FLOAT | Fuel consumed (from `fuel_consumption`) |
| `avg_speed_kmh` | FLOAT | Average speed during the event |
| `avg_rpm` | FLOAT | Average RPM during the event |
| `event_mileage_kmpl`| FLOAT | Final, unified mileage (km/L) for the event |
| `is_invalid_mileage_flag` | BOOLEAN | `True` if the final mileage is invalid. |
| `event_date` | DATE | The date the event occurred |

## 6.2 Output 2: Daily Summary Table
**Proposed Table Name:** `clutch_riding_daily_summary`
**Description:** Contains one row per vehicle per day, optimized for dashboards and reporting.

| Column Name | Data Type | Description |
|---|---|---|
| `analysis_date` | DATE | Date of analysis |
| `uniqueid` | VARCHAR | Vehicle identifier |
| `overall_distance_km`| FLOAT | Total distance traveled |
| `clutch_riding_mileage_kmpl`| FLOAT | Mileage during valid clutch riding events |
| `normal_riding_mileage_kmpl`| FLOAT | Mileage during valid normal riding events |
| `mileage_degradation_pct`| FLOAT | Percentage drop in mileage due to clutch riding |
| `is_short_duration_cycle` | BOOLEAN | `True` if total cycle duration is < 60 seconds |
| `is_short_distance_cycle` | BOOLEAN | `True` if total cycle distance is < 0.1 km |
| `has_data_gap_flag` | BOOLEAN | `True` if time between packets exceeded 45 seconds |
| `has_odometer_reset_flag`| BOOLEAN | `True` if the odometer reading went backward |

<div style="page-break-after: always;"></div>

# 7. IMPLEMENTATION DETAILS

## 7.1 Key Dependencies
- Python 3.x
- Pandas
- SQLAlchemy
- Psycopg2
- TQDM

## 7.2 Execution Strategy
The process is run within a Jupyter Notebook (`clutch_riding_production_7days.ipynb`). It uses a `ThreadPoolExecutor` to parallelize the processing of each cycle. A JSON file is used to track the processing status of each day.

## 7.3 Data Destination
The final dataframes are saved to CSV files. A separate process will be responsible for loading these CSVs into the final database tables.

<div style="page-break-after: always;"></div>

# 8. Customer-Facing Reporting

This section outlines the user interface (UI) and user experience (UX) for presenting the clutch riding analysis to the customer.

## 8.1 User Flow
The customer will navigate to the feature via the main application dashboard:
**`Dashboard` -> `Driver Behaviour` -> `Clutch Riding Analysis`**

Upon selecting the feature, the customer will be presented with a multi-level report.

## 8.2 Level 1: Fleet Summary
The top of the page will display key performance indicators (KPIs) for the entire fleet over the selected time period.

---
**Clutch Riding Fleet Overview**

| Total Vehicles Analyzed | Vehicles with Clutch Riding | Total Fuel Loss (Liters) | Estimated Savings Possible (INR) |
| :--- | :--- | :--- | :--- |
| 1,500 | 850 (57%) | 12,350 L | ₹ 1,111,500 |

---

## 8.3 Level 2: Vehicle Ranking Table
Below the fleet summary, a table will rank vehicles to identify the most significant offenders.

**Ranking Logic:** Vehicles will be sorted in descending order by the **total duration of "Long" and "Very Long" clutch riding events**. This prioritizes vehicles with the most severe and impactful clutch riding habits.

**Vehicle Performance Ranking**

| Rank | Vehicle ID | Total Clutch Riding (Hours) | Long/Very Long Events (Count) | Fuel Wasted (Liters) |
| :--- | :--- | :--- | :--- | :--- |
| 1 | VEH-102 | 15.2 | 45 | 150.5 |
| 2 | VEH-045 | 12.8 | 32 | 125.1 |
| 3 | VEH-119 | 11.5 | 25 | 110.8 |
| ... | ... | ... | ... | ... |

## 8.4 Level 3: Cycle-Level Drill-Down
Clicking on a vehicle row in the ranking table will expand to show a detailed, cycle-by-cycle breakdown for that vehicle.

**Cycle Analysis for Vehicle: VEH-102**

| Cycle Date | Cycle Start Time | Mileage Degradation | Potential Savings (INR) | Data Quality |
| :--- | :--- | :--- | :--- | :--- |
| 2026-02-05 | 08:15 AM | 35.2% | ₹ 85.50 | None |
| 2026-02-05 | 11:30 AM | 15.8% | ₹ 42.10 | None |
| 2026-02-04 | 04:45 PM | 41.0% | ₹ 105.20 | `HAS_DATA_GAP`|
| ... | ... | ... | ... | ... |
