---
title: "Clutch Riding Event-Based Analysis System"
subtitle: "Technical Product Requirements Document"
author: "Data Science Team"
date: "February 5, 2026"
version: "2.0"
classification: "Internal Use"
---

<div style="page-break-after: always;"></div>

# TABLE OF CONTENTS

1. [Document Overview](#1-document-overview)
2. [Business Objective](#2-business-objective)
3. [System Overview](#3-system-overview)
4. [Data Sources](#4-data-sources)
5. [Processing Logic](#5-processing-logic)
6. [Output Specification](#6-output-specification)
7. [Implementation Details](#7-implementation-details)

<div style="page-break-after: always;"></div>

---

# 1. DOCUMENT OVERVIEW

## 1.1 Purpose
This document specifies the technical requirements for an automated system that processes engine cycle data to generate a persistent, event-level table of clutch riding and normal riding behavior. This enables detailed, historical analysis of fuel efficiency and driver habits.

## 1.2 Scope
- **In Scope:** Processing of historical engine cycles, detection of clutch and normal riding events, and permanent storage of these events in a new database table.

## 1.3 Definitions

| Term | Definition |
|------|------------|
| **Engine Cycle** | The period from when a vehicle's engine is turned on to when it is turned off. |
| **Clutch Riding** | Occurs when speed is > 20 km/h, the clutch is pressed, and the accelerator is pressed (partially or completely). |
| **Riding Event** | A continuous period of time where a vehicle is either "clutch riding" or "normal riding". |
| **Event-Level Table** | A database table where each row represents a single, distinct riding event. |

<div style="page-break-after: always;"></div>

---

# 2. BUSINESS OBJECTIVE

## 2.1 Problem Statement
Clutch riding leads to poor fuel efficiency, resulting in unnecessary fuel costs and potential long-term vehicle wear. Analyzing this behavior requires a structured, event-based dataset that does not currently exist, making historical analysis and model training difficult.

## 2.2 Solution
Develop an automated data processing pipeline that:
1.  Identifies both clutch riding and normal riding events from raw vehicle telemetry.
2.  Calculates detailed metrics for each event (duration, distance, fuel, mileage, etc.).
3.  Creates two structured dataframes: a granular event-level summary and a cycle-level summary.
4.  Persists the granular event-level data into a new, permanent database table for analytical use.

## 2.3 Success Metrics
The success of this project will be measured by its ability to produce a dataset that is robust and enables clear, data-driven insights.

- **Data Robustness:** The pipeline must be able to successfully process any specified date range of historical data and populate the `clutch_riding_events` table.
- **Hypothesis Validation:** The final dataset must clearly demonstrate a statistically significant negative correlation between clutch riding and fuel efficiency.
- **Actionable Insight Generation:** The data must confirm that for "Long" and "Very Long" duration clutch riding events, mileage is worse than normal driving mileage in over 80% of observed cases.

<div style="page-break-after: always;"></div>

---

# 3. SYSTEM OVERVIEW

## 3.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                PostgreSQL Database                      │
│           (obd_data_history, engineoncycles)            │
└────────────────────┬────────────────────────────────────┘
                     │
                     │ SQL Queries (per engine cycle)
                     ▼
┌─────────────────────────────────────────────────────────┐
│                                                         │
│          Data Processing Pipeline (Jupyter)             │
│          (I/O and CPU-bound parallel workers)           │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │ 1. Fetch OBD data for each cycle                │   │
│  │ 2. Detect Clutch vs. Normal riding states       │   │
│  │ 3. Group states into discrete Events            │   │
│  │ 4. Generate Event-Level & Cycle-Level tables    │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└────────────────────┬────────────────────────────────────┘
                     │
                     │ Pandas DataFrame (combined_events)
                     ▼
┌─────────────────────────────────────────────────────────┐
│                                                         │
│                PostgreSQL Database                      │
│           (NEW: clutch_riding_events table)             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## 3.2 Processing Flow

**Step 1:** Fetch target engine cycles from the `engineoncycles` table in `OS_DB`.
**Step 2:** For each engine cycle, fetch the corresponding time-series OBD data from the `obd_data_history` table in `tracking_db` using the `uniqueid`, `cycle_start_ts`, and `cycle_end_ts`.
**Step 3:** Process the data for each cycle (in parallel):
&nbsp;&nbsp;&nbsp;&nbsp;→ Identify `is_clutch_riding` status for each packet.
&nbsp;&nbsp;&nbsp;&nbsp;→ Group consecutive packets with the same status into events.
&nbsp;&nbsp;&nbsp;&nbsp;→ For each event, calculate metrics (duration, distance, fuel, etc.).
&nbsp;&nbsp;&nbsp;&nbsp;→ For each cycle, calculate aggregate summary statistics.
**Step 4:** Concatenate all event-level data from all cycles into a single `combined_events` dataframe.
**Step 5:** Save the `combined_events` dataframe to the new `clutch_riding_events` database table.

<div style="page-break-after: always;"></div>

---

# 4. DATA SOURCES

## 4.1 Input Table 1: OBD Data

**Database:** PostgreSQL `tracking_db`
**Table:** `public.obd_data_history`

### 4.1.1 Relevant Schema

| Column Name | Data Type | Description |
|---|---|---|
| `uniqueid` | VARCHAR | Vehicle identifier |
| `ts` | INTEGER | Unix timestamp |
| `vehiclespeed`| NUMERIC | Current vehicle speed in km/h |
| `rpm` | NUMERIC | Engine revolutions per minute |
| `clutch_switch_status`| VARCHAR | Status of the clutch pedal ('Pressed', 'Released') |
| `accelerator_pedal_pos`| NUMERIC | Throttle position percentage |
| `fuel_consumption`| NUMERIC | Cumulative fuel used by the vehicle |
| `fuel_rate` | NUMERIC | Instantaneous fuel rate |
| `obddistance` | NUMERIC | Cumulative distance from odometer |

## 4.2 Input Table 2: Engine Cycles
**Database:** `OS_DB`
**Table:** `public.engineoncycles`

### 4.2.1 Relevant Schema
| Column Name | Data Type | Description |
|---|---|---|
| `cycle_id` | VARCHAR | Unique identifier for the engine cycle |
| `uniqueid` | VARCHAR | Vehicle identifier, used to join with OBD data |
| `cycle_start_ts`| INTEGER | Start timestamp for filtering OBD data |
| `cycle_end_ts` | INTEGER | End timestamp for filtering OBD data |

<div style="page-break-after: always;"></div>

---

# 5. PROCESSING LOGIC

## 5.1 Event Definition
An **event** is a continuous period of driving behavior. The system identifies two types of events:
1.  **Clutch Riding Event:** A period where the driver is engaging the clutch unnecessarily.
2.  **Normal Riding Event:** Any driving period that is not a clutch riding event.

An event ends and a new one begins ONLY when the `is_clutch_riding` status changes.

## 5.2 Clutch Riding Detection Criteria

A data packet is flagged as `is_clutch_riding = TRUE` if **ALL** three of the following conditions are met:

```python
is_clutch_riding = (
    clutch_switch_status == 'Pressed'
    AND vehiclespeed > 20
    AND accelerator_pedal_pos > 2
)
```

## 5.3 Event Grouping

Consecutive data packets are grouped into a single event using the following logic. A new group (and thus a new event) is started whenever the boolean value of `is_clutch_riding` changes from the previous data packet.

```python
# This creates a unique ID for each continuous block of True or False
df['event_group'] = (df['is_clutch_riding'] != df['is_clutch_riding'].shift()).cumsum()
```

## 5.4 Fuel Calculation
Fuel consumption for each event is calculated using two methods for robustness and comparison:
1.  **From `fuel_rate`:** `(fuel_rate * 5 / 3600)`. This is considered more accurate for short events.
2.  **From `fuel_consumption`:** Calculating the `diff()` on the cumulative counter.

The final analysis primarily relies on the `fuel_rate` calculation.

```python
# Method 1: From instantaneous fuel_rate
# Assumes a 5-second interval between packets
df['fuel_from_rate'] = (df['fuel_rate'] * 5 / 3600).fillna(0)

# Method 2: From cumulative fuel_consumption
df['fuel_from_consumption'] = df['fuel_consumption'].diff().fillna(0)

# Basic validation for the diff method
df.loc[df['fuel_from_consumption'] < 0, 'fuel_from_consumption'] = 0
```

## 5.5 Event Aggregation
Once packets are grouped by `event_group`, each group is aggregated to create a single row in the final events table. This involves summing interval-based metrics and averaging point-in-time metrics.
```python
# Simplified example of aggregating a single group
event = {
    'event_type': 'clutch_riding',
    'event_start_ts': group_df['ts'].iloc[0],
    'event_end_ts': group_df['ts'].iloc[-1],
    'event_duration_sec': group_df['event_duration'].sum(),
    'event_distance_m': group_df['event_distance'].sum(),
    'avg_speed_kmh': group_df['vehiclespeed'].mean()
}
```

<div style="page-break-after: always;"></div>

---

# 6. OUTPUT SPECIFICATION

The pipeline produces two distinct, persistent tables in the database, designed for different analytical needs.

## 6.1 Primary Output 1: Event-Level Table
This table provides the most granular data, with one row for every detected riding event. It is designed for deep, exploratory analysis and model training.

**Proposed Table Name:** `clutch_riding_events`

### 6.1.1 Table Schema
| Column Name | Data Type | Description |
|---|---|---|
| `cycle_id` | VARCHAR | Foreign key to the engine cycle |
| `uniqueid` | VARCHAR | Vehicle identifier |
| `event_type` | VARCHAR | 'clutch_riding' or 'normal_riding' |
| `event_start_ts`| INTEGER | Start timestamp of the event |
| `event_end_ts` | INTEGER | End timestamp of the event |
| `event_duration_sec`| FLOAT | Duration of the event in seconds |
| `event_distance_m`| FLOAT | Distance covered during the event in meters |
| `event_fuel_from_rate_liters` | FLOAT | Fuel consumed during the event (calculated from `fuel_rate`) |
| `avg_speed_kmh` | FLOAT | Average speed during the event |
| `avg_rpm` | FLOAT | Average RPM during the event |
| `consecutive_duration_category`| VARCHAR | 'Short (<10s)', 'Medium (10-30s)', etc. |
| `event_mileage_from_rate_kmpl` | FLOAT | Mileage (km/L) for the event |
| `event_date` | DATE | The date the event occurred |

## 6.2 Primary Output 2: Daily Summary Table
This table provides a high-level summary for each vehicle for each day, aggregating all of its cycle data. It is optimized for dashboards and daily reporting.

**Proposed Table Name:** `clutch_riding_daily_summary`

### 6.2.1 Table Schema
| Column Name | Data Type | Description | Unit |
|---|---|---|---|
| `analysis_date` | DATE | Date of analysis | YYYY-MM-DD |
| `uniqueid` | VARCHAR | Vehicle identifier | - |
| `overall_distance_km` | FLOAT | Total distance traveled | km |
| `overall_fuel_liters` | FLOAT | Total fuel consumed | liters |
| `overall_mileage_kmpl` | FLOAT | Overall fuel efficiency | km/L |
| `clutch_riding_distance_km` | FLOAT | Distance during clutch riding | km |
| `clutch_riding_fuel_liters` | FLOAT | Fuel during clutch riding | liters |
| `clutch_riding_mileage_kmpl` | FLOAT | Mileage during clutch riding | km/L |
| `normal_riding_distance_km` | FLOAT | Distance during normal riding | km |
| `normal_riding_fuel_liters` | FLOAT | Fuel during normal riding | liters |
| `normal_riding_mileage_kmpl` | FLOAT | Mileage during normal riding | km/L |
| `clutch_riding_distance_pct` | FLOAT | Percentage of distance while clutch riding | % |
| `clutch_riding_fuel_pct` | FLOAT | Percentage of fuel consumed by clutch riding | % |
| `fuel_savings_possible_liters` | FLOAT | Potential fuel savings | liters |
| `monetary_savings_inr` | FLOAT | Potential cost savings | ₹ |


---

# 7. IMPLEMENTATION DETAILS

## 7.1 Key Dependencies
- Python 3.x
- Pandas
- SQLAlchemy
- Psycopg2 (for PostgreSQL connection)
- TQDM

## 7.2 Execution Strategy
The process is designed to be run within a Jupyter Notebook (`clutch_riding_production_7days.ipynb`). It uses a `ThreadPoolExecutor` to parallelize both the fetching of OBD data and the CPU-bound processing of each cycle's data to improve performance. A JSON file is used to track the processing status of each day to prevent reprocessing.

## 7.3 Data Destination
The final `combined_events` dataframe should be written to the `clutch_riding_events` table. A mechanism like `pandas.DataFrame.to_sql` with the `if_exists='append'` parameter should be used to incrementally add new events to the table.

---
**END OF DOCUMENT**
