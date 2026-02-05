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
- **Out of Scope:** Real-time monitoring, driver-facing dashboards (covered in a separate product PRD).

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
-   Successfully process 7 days of historical data for the target fleet.
-   Populate the new `clutch_riding_events` table with event data.
-   The resulting data must be sufficient to power downstream analysis, including hypothesis testing and dashboard creation.

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

**Step 1:** Load engine cycle data for the target date range from a fallback file (`engineoncycles_temp.csv`).
**Step 2:** For each engine cycle (in parallel):
&nbsp;&nbsp;&nbsp;&nbsp;→ Fetch time-series OBD data from `obd_data_history`, including `fuel_rate`.
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

**Source:** Fallback File
**File:** `engineoncycles_temp.csv`

### 4.2.1 Relevant Schema

| Column Name | Data Type | Description |
|---|---|---|
| `cycle_id` | VARCHAR | Unique identifier for the engine cycle |
| `uniqueid` | VARCHAR | Vehicle identifier |
| `cycle_start_ts` | INTEGER | Unix timestamp for the start of the cycle |
| `cycle_end_ts` | INTEGER | Unix timestamp for the end of the cycle |

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

<div style="page-break-after: always;"></div>

---

# 6. OUTPUT SPECIFICATION

## 6.1 Primary Output: Database Table

The primary output of this pipeline is a new, persistent table in the database.

**Database:** PostgreSQL `tracking_db` (or other analytical DB)
**Proposed Table Name:** `clutch_riding_events`

### 6.1.1 Table Schema

This table will contain one row for every detected riding event.

| Column Name | Data Type | Description |
|---|---|---|
| `cycle_id` | VARCHAR | Foreign key to the engine cycle |
| `uniqueid` | VARCHAR | Vehicle identifier |
| `event_type` | VARCHAR | 'clutch_riding' or 'normal_riding' |
| `event_start_ts`| INTEGER | Start timestamp of the event |
| `event_end_ts` | INTEGER | End timestamp of the event |
| `event_duration_sec`| FLOAT | Duration of the event in seconds |
| `event_distance_m`| FLOAT | Distance covered during the event in meters |
| `event_fuel_from_rate_liters` | FLOAT | Fuel consumed (from fuel_rate) |
| `event_fuel_from_consumption_liters`| FLOAT | Fuel consumed (from cumulative counter) |
| `avg_speed_kmh` | FLOAT | Average speed during the event |
| `avg_rpm` | FLOAT | Average RPM during the event |
| `consecutive_duration_category`| VARCHAR | 'Short (<10s)', 'Medium (10-30s)', etc. |
| `event_mileage_from_rate_kmpl` | FLOAT | Mileage (km/L) for the event |
| `event_date` | DATE | The date the event occurred |

## 6.2 Secondary Output: In-Memory Dataframe

A second dataframe, `clutch_riding_cycle_summary`, is generated for analytical purposes within the notebook but is not designated for persistent storage in this version. It aggregates event data to the cycle level to calculate metrics like `mileage_degradation_pct`.

<div style="page-break-after: always;"></div>

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
