---
title: "Clutch Riding Analysis System"
subtitle: "Technical Product Requirements Document"
author: "Data Science Team"
date: "January 19, 2026"
version: "1.0"
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
This document specifies the technical requirements for an automated system that generates day-wise clutch riding statistics for fleet vehicles, enabling fuel efficiency analysis and cost savings identification.

## 1.2 Scope
- **In Scope:** Daily analysis of clutch riding behavior and fuel consumption impact
- **Out of Scope:** Vehicle health metrics, real-time monitoring, driver behavior patterns

## 1.3 Definitions

| Term | Definition |
|------|------------|
| **Clutch Riding** | Operating the clutch pedal while accelerating at speeds above 20 km/h |
| **Event-Based Data** | Time-series data transformed into interval-based measurements |
| **OBD** | On-Board Diagnostics - vehicle sensor telemetry system |
| **Mileage** | Fuel efficiency measured in kilometers per liter (km/L) |

<div style="page-break-after: always;"></div>

---

# 2. BUSINESS OBJECTIVE

## 2.1 Problem Statement
Clutch riding leads to poor fuel efficiency, resulting in unnecessary fuel costs. Currently, there is no automated system to quantify this impact across the fleet.

## 2.2 Solution
Develop an automated daily analysis system that:
1. Identifies clutch riding events from vehicle telemetry data
2. Calculates fuel consumption during clutch riding vs. normal riding
3. Quantifies potential fuel and monetary savings
4. Generates actionable reports for fleet management

## 2.3 Success Metrics
- Process 85%+ of active fleet vehicles daily
- Generate statistics within 10 minutes
- Identify vehicles with >20% clutch riding for driver training
- Calculate potential savings per vehicle per day

<div style="page-break-after: always;"></div>

---

# 3. SYSTEM OVERVIEW

## 3.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│                PostgreSQL Database                      │
│           (obd_data_history table)                      │
│                                                         │
└────────────────────┬────────────────────────────────────┘
                     │
                     │ SQL Queries (per vehicle, per day)
                     ▼
┌─────────────────────────────────────────────────────────┐
│                                                         │
│          Data Processing Pipeline                       │
│          (10 parallel workers)                          │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │ 1. Forward Fill (handle NULLs)                  │   │
│  │ 2. Event Feature Engineering (next - current)   │   │
│  │ 3. Clutch Riding Detection (flag events)        │   │
│  │ 4. Statistics Calculation (aggregate metrics)   │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└────────────────────┬────────────────────────────────────┘
                     │
                     │ Excel Output Files
                     ▼
┌─────────────────────────────────────────────────────────┐
│                                                         │
│         Final Output Tables                             │
│   - Daily Statistics                                    │
│   - Savings Summary                                     │
│   - Quality Reports                                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## 3.2 Processing Flow

**Step 1:** Fetch vehicle IDs active on target date
**Step 2:** For each vehicle (parallel processing):
&nbsp;&nbsp;&nbsp;&nbsp;→ Fetch time-series data
&nbsp;&nbsp;&nbsp;&nbsp;→ Forward fill cumulative metrics
&nbsp;&nbsp;&nbsp;&nbsp;→ Create event-based features
&nbsp;&nbsp;&nbsp;&nbsp;→ Detect clutch riding events
&nbsp;&nbsp;&nbsp;&nbsp;→ Calculate statistics
**Step 3:** Aggregate results and generate output files

<div style="page-break-after: always;"></div>

---

# 4. DATA SOURCES

## 4.1 Input Table

**Database:** PostgreSQL `tracking_db`
**Schema:** `public`
**Table:** `obd_data_history`

### 4.1.1 Table Schema

| Column Name | Data Type | Description | Unit | Sample Value |
|------------|-----------|-------------|------|--------------|
| `uniqueid` | VARCHAR | Vehicle identifier | - | VEH001 |
| `ts` | INTEGER | Unix timestamp | seconds | 1705334400 |
| `vehiclespeed` | NUMERIC | Current speed | km/h | 45.5 |
| `rpm` | NUMERIC | Engine RPM | rev/min | 1850 |
| `clutch_switch_status` | VARCHAR | Clutch pedal status | - | Pressed |
| `accelerator_pedal_pos` | NUMERIC | Throttle position | % | 35.2 |
| `fuel_consumption` | NUMERIC | Cumulative fuel used | liters | 125.8 |
| `obddistance` | NUMERIC | Cumulative odometer | meters | 185420 |

### 4.1.2 Data Characteristics

- **Update Frequency:** Real-time (1-60 second intervals)
- **Data Volume:** ~1,000,000 records per day (fleet-wide)
- **Records per Vehicle:** 50 - 5,000 per day
- **Cumulative Columns:** `fuel_consumption`, `obddistance` (monotonically increasing)
- **Sparse Data:** NULL values when vehicle is stationary

### 4.1.3 SQL Query Pattern

```sql
-- Fetch vehicle data for one day
SELECT
    uniqueid,
    ts,
    vehiclespeed,
    rpm,
    clutch_switch_status,
    accelerator_pedal_pos,
    fuel_consumption,
    obddistance
FROM public.obd_data_history
WHERE ts >= :day_start_ts
  AND ts < :day_end_ts
  AND uniqueid = :vehicle_id
ORDER BY ts ASC;
```

<div style="page-break-after: always;"></div>

---

# 5. PROCESSING LOGIC

## 5.1 Data Preprocessing

### 5.1.1 Forward Filling

**Purpose:** Handle NULL values in cumulative sensor readings

**Logic:**
```python
# When vehicle is stationary, cumulative sensors may return NULL
# Forward fill propagates the last known value
df['obddistance'] = df['obddistance'].ffill()
df['fuel_consumption'] = df['fuel_consumption'].ffill()
```

**Example:**

| Timestamp | obddistance (raw) | obddistance (filled) |
|-----------|-------------------|----------------------|
| 10:00:00 | 50000 | 50000 |
| 10:00:30 | NULL | 50000 ← filled |
| 10:01:00 | NULL | 50000 ← filled |
| 10:01:30 | 50250 | 50250 |

## 5.2 Event-Based Feature Engineering

### 5.2.1 Concept
Transform time-series data into interval-based measurements by calculating changes between consecutive timestamps.

### 5.2.2 Formulas

For each row `i`, calculate the change until the next row `i+1`:

```
event_distance[i] = obddistance[i+1] - obddistance[i]      (meters)
event_fuel[i] = fuel_consumption[i+1] - fuel_consumption[i]  (liters)
event_duration[i] = ts[i+1] - ts[i]                         (seconds)
```

### 5.2.3 Example Calculation

| Row | ts | obddistance | fuel | event_distance | event_fuel |
|-----|---------|-------------|--------|----------------|------------|
| 0 | 10:00:00 | 50000 | 10.50 | 250 | 0.05 |
| 1 | 10:00:30 | 50250 | 10.55 | 180 | 0.03 |
| 2 | 10:01:00 | 50430 | 10.58 | 320 | 0.06 |
| 3 | 10:01:30 | 50750 | 10.64 | - | - |

**Note:** Last row dropped (no "next" reading available)

### 5.2.4 Data Validation Rules

| Validation | Condition | Action | Rationale |
|------------|-----------|--------|-----------|
| Odometer Reset | event_distance < 0 | Set to 0 | Negative distance is impossible |
| Excessive Distance | event_distance > 50,000m | Set to 0 | Single event can't exceed 50 km |
| Unrealistic Speed | distance > (200 km/h × time × 1.5) | Set to 0 | Data transmission error |
| Negative Fuel | event_fuel < 0 | Set to 0 | Sensor error or refill |

## 5.3 Clutch Riding Detection

### 5.3.1 Detection Criteria

An event is flagged as **clutch riding** if **ALL** three conditions are met:

```python
is_clutch_riding = (
    clutch_switch_status == 'Pressed'      # Clutch pedal engaged
    AND vehiclespeed > 20                   # Vehicle in motion
    AND accelerator_pedal_pos > 2           # Driver applying throttle
)
```

### 5.3.2 Threshold Rationale

| Threshold | Value | Rationale |
|-----------|-------|-----------|
| Speed | > 20 km/h | Below 20 km/h indicates starting from stop (normal clutch usage) |
| Accelerator | > 2% | Below 2% indicates coasting or deceleration (normal) |
| Clutch | Pressed | Obvious requirement for clutch riding |

### 5.3.3 Excluded Scenarios

The following are **NOT** considered clutch riding:
- Starting from stop (speed < 20 km/h)
- Coasting with clutch pressed (accelerator < 2%)
- Normal gear changes at any speed
- Stationary with engine running

## 5.4 Statistics Calculation

### 5.4.1 Metric Categories

Statistics are calculated for three categories:

1. **Overall** - All events (clutch + normal)
2. **Clutch Riding** - Events where `is_clutch_riding = TRUE`
3. **Normal Riding** - Events where `is_clutch_riding = FALSE`

### 5.4.2 Metrics Calculated

For each category:

| Metric | Formula | Unit |
|--------|---------|------|
| Total Distance | `sum(event_distance) / 1000` | km |
| Total Fuel | `sum(event_fuel)` | liters |
| Mileage | `total_distance_km / total_fuel_liters` | km/L |
| Duration | `sum(event_duration)` | seconds |

### 5.4.3 Savings Calculation

```python
# Step 1: Calculate ideal fuel consumption
fuel_should_have_used = clutch_riding_distance_km / normal_riding_mileage_kmpl

# Step 2: Calculate wasted fuel
fuel_wasted = clutch_riding_fuel_liters - fuel_should_have_used

# Step 3: Calculate savings potential
fuel_savings_possible_liters = max(0, fuel_wasted)
monetary_savings_inr = fuel_savings_possible_liters × 100  # ₹100/liter
```

**Interpretation:** Amount of fuel (and money) that could be saved if the driver maintained normal riding efficiency during clutch riding segments.

<div style="page-break-after: always;"></div>

---

# 6. OUTPUT SPECIFICATION

## 6.1 Final Output Table

### Table Name: `clutch_riding_daily_summary`

### 6.1.1 Schema Definition

| Column Name | Data Type | Description | Unit | Example |
|------------|-----------|-------------|------|---------|
| **Identifiers** |
| `analysis_date` | DATE | Date of analysis | YYYY-MM-DD | 2026-01-15 |
| `uniqueid` | VARCHAR | Vehicle identifier | - | VEH001 |
| **Overall Metrics** |
| `overall_distance_km` | FLOAT | Total distance traveled | km | 185.5 |
| `overall_fuel_liters` | FLOAT | Total fuel consumed | liters | 15.2 |
| `overall_mileage_kmpl` | FLOAT | Overall fuel efficiency | km/L | 12.2 |
| **Clutch Riding Metrics** |
| `clutch_riding_distance_km` | FLOAT | Distance during clutch riding | km | 28.4 |
| `clutch_riding_fuel_liters` | FLOAT | Fuel during clutch riding | liters | 3.5 |
| `clutch_riding_mileage_kmpl` | FLOAT | Mileage during clutch riding | km/L | 8.1 |
| **Normal Riding Metrics** |
| `normal_riding_distance_km` | FLOAT | Distance during normal riding | km | 157.1 |
| `normal_riding_fuel_liters` | FLOAT | Fuel during normal riding | liters | 11.7 |
| `normal_riding_mileage_kmpl` | FLOAT | Mileage during normal riding | km/L | 13.4 |
| **Comparison Metrics** |
| `clutch_riding_distance_pct` | FLOAT | Percentage of distance | % | 15.3 |
| `clutch_riding_fuel_pct` | FLOAT | Percentage of fuel | % | 23.0 |
| **Savings Potential** |
| `fuel_savings_possible_liters` | FLOAT | Potential fuel savings | liters | 1.38 |
| `monetary_savings_inr` | FLOAT | Potential cost savings | ₹ | 138.00 |

### 6.1.2 Sample Data

```
analysis_date | uniqueid | overall_distance_km | overall_fuel_liters | overall_mileage_kmpl |
              |          |                     |                     |                      |
2026-01-15    | VEH001   | 185.5               | 15.2                | 12.2                 |
2026-01-15    | VEH002   | 165.0               | 18.5                | 8.9                  |
2026-01-15    | VEH003   | 142.8               | 12.1                | 11.8                 |

clutch_riding_distance_km | clutch_riding_fuel_liters | clutch_riding_mileage_kmpl |
                          |                           |                            |
28.4                      | 3.5                       | 8.1                        |
62.3                      | 9.8                       | 6.4                        |
5.2                       | 0.6                       | 8.7                        |

normal_riding_distance_km | normal_riding_fuel_liters | normal_riding_mileage_kmpl |
                          |                           |                            |
157.1                     | 11.7                      | 13.4                       |
102.7                     | 8.7                       | 11.8                       |
137.6                     | 11.5                      | 12.0                       |

clutch_riding_distance_pct | clutch_riding_fuel_pct | fuel_savings_possible_liters | monetary_savings_inr |
                           |                        |                              |                      |
15.3                       | 23.0                   | 1.38                         | 138.00               |
37.8                       | 53.0                   | 4.52                         | 452.00               |
3.6                        | 5.0                    | 0.17                         | 17.00                |
```

## 6.2 Output Files

### 6.2.1 Primary Outputs

| File Name | Description | Use Case |
|-----------|-------------|----------|
| `clutch_riding_statistics.xlsx` | Complete daily statistics | Detailed analysis |
| `mileage_summary_filtered_10km.xlsx` | Vehicles with >10 km distance | Statistical reliability |
| `dqm_report.xlsx` | Data quality metrics | Quality assurance |
| `failed_vehicles.xlsx` | Processing failures | Debugging |

### 6.2.2 File Format
- **Format:** Microsoft Excel (.xlsx)
- **Encoding:** UTF-8
- **Structure:** One row per vehicle per day
- **Generation Time:** Daily at 03:00 AM
- **Retention Period:** 90 days

## 6.3 Data Quality Filters

Vehicles are **excluded** from the final table if they meet any of these criteria:

| Filter | Threshold | Reason |
|--------|-----------|--------|
| Sample Size | < 50 records | Insufficient data points |
| Fuel Data Quality | > 60% NULL values | Cannot calculate mileage |
| Clutch Data Quality | > 50% NULL values | Cannot detect clutch riding |
| Distance Traveled | < 10 km | Not statistically significant |

<div style="page-break-after: always;"></div>

---

# 7. IMPLEMENTATION DETAILS

## 7.1 Technical Configuration

### 7.1.1 System Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| **Processing** |
| Parallel Workers | 10 | Concurrent vehicle processing threads |
| Processing Mode | Per-vehicle, per-day | Isolated processing per vehicle |
| Date Order | Newest to oldest | Recent data prioritized |
| **Database** |
| Connection Pool | 20 connections | Base connection pool size |
| Max Overflow | 30 connections | Additional connections during peak |
| Query Timeout | 30 seconds | Maximum query execution time |
| Retry Attempts | 3 | Failed query retry count |
| Retry Delay | 2 seconds | Base delay (exponential backoff) |
| **Execution** |
| Scheduled Time | 03:00 AM daily | Low database load period |
| Target Duration | < 10 minutes | Expected completion time |
| Max Duration | 20 minutes | Alert threshold |

### 7.1.2 Resource Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU Cores | 10 | 12 |
| Memory (RAM) | 8 GB | 16 GB |
| Storage | 1 GB/day | 100 GB total |
| Network | 100 Mbps | 1 Gbps |

## 7.2 Performance Expectations

### 7.2.1 Processing Metrics

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| Success Rate | > 85% | < 85% | < 70% |
| Processing Time | < 5 min | > 10 min | > 20 min |
| Vehicles per Day | 1,000 - 5,000 | - | - |
| Average Time per Vehicle | 0.5 - 2 sec | - | - |

### 7.2.2 Expected Throughput

**For 1,000 vehicles:**
- Fetch vehicle IDs: ~5 seconds
- Process vehicles (parallel): ~3 minutes
- Generate outputs: ~5 seconds
- **Total:** ~3-5 minutes

## 7.3 Key Formulas Reference

```python
# Event calculation
event_distance[i] = obddistance[i+1] - obddistance[i]
event_fuel[i] = fuel_consumption[i+1] - fuel_consumption[i]
event_duration[i] = ts[i+1] - ts[i]

# Clutch riding detection
is_clutch_riding = (
    (clutch_switch_status == 'Pressed') &
    (vehiclespeed > 20) &
    (accelerator_pedal_pos > 2)
)

# Mileage calculation
mileage_kmpl = (sum(event_distance) / 1000) / sum(event_fuel)

# Savings calculation
fuel_should_have_used = clutch_distance_km / normal_mileage_kmpl
fuel_wasted = clutch_fuel_liters - fuel_should_have_used
savings_liters = max(0, fuel_wasted)
savings_inr = savings_liters × 100
```

## 7.4 Success Criteria

### Per Vehicle Analysis
- ✓ Daily statistics generated for each vehicle
- ✓ Clutch riding distance, fuel, and mileage calculated
- ✓ Savings potential quantified
- ✓ Data stored in final output table

### Fleet-Wide Execution
- ✓ Minimum 85% of vehicles successfully processed
- ✓ All output files generated without errors
- ✓ Processing completes within 10 minutes
- ✓ No critical data quality issues

<div style="page-break-after: always;"></div>

---

# APPENDIX

## A. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-19 | Data Science Team | Initial release |

## B. References

- Source Code: `clutch_riding_done_structured.ipynb`
- Database: PostgreSQL `tracking_db.public.obd_data_history`
- Output Directory: `./outputs/`

## C. Contact Information

| Role | Contact |
|------|---------|
| Technical Lead | Data Science Team |
| Product Owner | Fleet Management |
| Database Admin | IT Infrastructure |

---

**Document Classification:** Internal Use
**Distribution:** Authorized Personnel Only
**Next Review Date:** 2026-07-19

---

**END OF DOCUMENT**
