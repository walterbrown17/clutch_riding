# Clutch Riding Module — Technical Documentation

**Project:** OBD Vehicle Health — `vehicleHealth_v2`
**Module path:** `api/driver_behaviour/clutch_riding/`
**Author:** Satyam Mandloi
**Date:** March 2026

---

## 1. Overview

This module detects and analyses **clutch riding behaviour** in commercial vehicles using raw OBD (On-Board Diagnostics) telematics data. Clutch riding is the habit of keeping the clutch partially or fully pressed while the vehicle is in motion — it increases clutch wear, raises fuel consumption, and reduces mileage.

The module is designed around a two-stage architecture:

| Stage | Description |
|---|---|
| **Detection** (per vehicle) | Fetch raw OBD packets → segment into engine cycles → detect clutch/normal riding events → merge noise → compute vehicle-level metrics |
| **Aggregation** (fleet-wide) | Read the stored events table → compute fleet-level KPIs → benchmark every vehicle → classify into 4 tiers |

---

## 2. File Structure

```
api/driver_behaviour/clutch_riding/
├── config.py         All detection thresholds and constants
├── logic.py          Core algorithms: DQM cleaning, event detection, merging, metrics
├── service.py        Pipeline orchestration: OBD fetch → logic → events ready for storage
├── aggregation.py    Fleet analytics and vehicle benchmarking on the stored events table
├── resolver.py       (pending) GraphQL resolver
└── schema.graphql    (pending) GraphQL type definitions
```

---

## 3. File-by-File Breakdown

### 3.1 `config.py` — Constants

Central store for all tunable thresholds. Any change to detection sensitivity or data quality limits is made here only.

| Constant | Value | Purpose |
|---|---|---|
| `CLUTCH_SPEED_THRESHOLD` | 20 km/h | Minimum vehicle speed to classify a packet as clutch riding |
| `CLUTCH_ACCEL_THRESHOLD` | 2 | Minimum accelerator pedal position to classify clutch riding |
| `DURATION_SHORT` | 10 s | Events < 10s are labelled "Short" |
| `DURATION_MEDIUM` | 30 s | Events 10–30s are labelled "Medium" |
| `DURATION_LONG` | 60 s | Events 30–60s are labelled "Long"; >= 60s = "Very Long" |
| `MAX_DISTANCE_M` | 5000 m | OBD distance delta above this per packet is treated as an outlier and zeroed |
| `MAX_FUEL_L` | 2 L | Fuel consumption delta above this per packet is treated as an outlier and zeroed |
| `HA_IMPOSSIBLE_SPEED_KMH` | 250 km/h | Speed at or above this is flagged as a sensor error |
| `DEFAULT_PACKET_INTERVAL_SEC` | 5 s | Assumed OBD packet interval, used to estimate fuel consumed from fuel rate |

---

### 3.2 `logic.py` — Core Algorithms

Contains the full per-cycle processing pipeline. All functions in this file operate on a single vehicle's data for a given time window.

#### A. `clean_and_flag_data(df, cycle_id) → (cleaned_df, dqm_flags)`

Cleans a raw OBD cycle DataFrame and produces Data Quality Management (DQM) flags.

**What it does:**
1. Checks for missing sensors — fuel, distance, RPM, clutch switch
2. Detects time gaps > 60 seconds (continuity issue)
3. Detects impossible speeds >= 250 km/h (sensor integrity issue)
4. Ensures all required columns exist, filling missing ones with 0
5. Normalises numeric columns (`vehiclespeed`, `rpm`, `accelerator_pedal_pos`, `lat`, `lng`)
6. Forward-fills cumulative OBD counters (`obddistance`, `fuel_consumption`) to handle intermittent dropouts

**DQM flags returned:**
```
fuel_missing, dist_missing, rpm_missing, clutch_missing,
large_time_gap_exists, impossible_speed_detected
```
These flags travel with every event row, providing full transparency on data quality.

---

#### B. `detect_clutch_events(df, cycle_id, dqm_flags) → list[dict]`

Detects clutch-riding and normal-riding events from cleaned OBD packets.

**Clutch riding condition (per packet):**
```
clutch_switch_status == 'Pressed'
AND  vehiclespeed > 20 km/h
AND  accelerator_pedal_pos > 2
```

**Steps:**
1. Compute per-packet deltas: duration, OBD distance, fuel consumed, fuel from rate
2. Compute Haversine GPS distance between consecutive packets for validation; zero out OBD distance outliers
3. Mark each packet as `clutch_riding` or `normal_riding` using the condition above
4. Group consecutive same-state packets into events
5. Skip events shorter than 2 seconds (noise)
6. For each event, record: type, timestamps, duration, distance, avg speed, avg RPM, severity category, fuel consumed, mileage, DQM flags

**Event dict structure:**
```python
{
  cycle_id, event_type,          # 'clutch_riding' | 'normal_riding'
  event_start_epoch, event_end_epoch,
  duration_sec, distance_km,
  avg_speed_kmh, avg_rpm,
  severity,                      # 'Short' | 'Medium' | 'Long' | 'Very Long'
  fuel_consumed_liters, mileage_kmpl,
  dqm_fuel_error, dqm_dist_error, dqm_speed_error, dqm_gap_error
}
```

---

#### C. `merge_events(df) → DataFrame`

Removes noise at event boundaries using iterative merging. Runs until no further merges are possible (handles cascades).

**Case 1 — Short clutch burst absorbed into normal riding:**
```
normal_riding (>= 30s) → clutch_riding (<= 10s) → normal_riding (>= 30s)
                       ↓
              merged → normal_riding
```

**Case 2 — Short normal gap absorbed into clutch riding:**
```
clutch_riding → normal_riding (<= 10s) → clutch_riding
                      ↓
             merged → clutch_riding
```

**Merge rules for the combined row:**
- Duration, distance, fuel: **summed**
- Speed, RPM: **duration-weighted average**
- Severity: **recalculated** from merged duration
- Mileage: **recalculated** from merged distance / fuel

---

#### D. `compute_metrics(df) → dict`

Computes a summary for one vehicle's merged events DataFrame.

**Mileage rule:** `sum(distance_km) / sum(fuel_consumed_liters)` using **only Medium, Long, Very Long** events. Short events (< 10s) are excluded from the mileage calculation because they are too brief to produce reliable fuel readings.

**Output:**
```python
{
  overall_distance_km,       overall_consumption_liters,  overall_mileage_kmpl,
  clutch_riding_distance_km, clutch_riding_consumption_liters, clutch_riding_mileage_kmpl,
  normal_riding_distance_km, normal_riding_consumption_liters, normal_riding_mileage_kmpl,
  clutch_riding_pct   # = (clutch_distance / overall_distance) * 100
}
```

---

### 3.3 `service.py` — Pipeline Orchestration

Ties everything together. A single call to `run_clutch_analysis()` fetches raw OBD data, runs the full logic pipeline, and returns merged events + metrics ready to be stored in the fundamental events table.

#### `fetch_obd_for_clutch(unique_id, from_time, to_time) → list[dict]`

SQL fetch from `obd_data_history` for the columns needed by clutch analysis:
```sql
SELECT uniqueid, ts, vehiclespeed, rpm, clutch_switch_status,
       accelerator_pedal_pos, lat, lng, obddistance, fuel_consumption, fuel_rate
FROM obd_data_history
WHERE uniqueid = %s AND ts >= %s AND ts <= %s
ORDER BY ts ASC
```
Uses the existing `tracking_db_pool` singleton and `fetch_data_with_column_keys` utility.

#### `run_clutch_analysis(...) → dict`

**Supports three cycle segmentation modes:**

| Mode | How | When to use |
|---|---|---|
| On-the-fly | Runs `preprocess_for_cycles` + `detect_cycles` from `core/engine_cycle` | Default; no pre-computed cycles needed |
| Stored cycles | Queries `public.engineoncycles` table | When cycles are already pre-computed and stored |
| Manual inject | Caller passes `manual_cycles_data` | Testing, notebooks, backfill scripts |

**Pipeline steps:**
```
1. Ingest raw OBD data (DB fetch or manual inject)
2. Segment into engine cycles (one of 3 modes above)
3. For each cycle:
     a. Slice the cycle's packets from full_df
     b. clean_and_flag_data()      → cleaned df + DQM flags
     c. detect_clutch_events()     → raw event list
4. merge_events()                  → noise-reduced event DataFrame
5. compute_metrics()               → vehicle-level KPI dict
6. Return { "events": [...], "metrics": {...} }
```

The returned `events` list is intended to be persisted into the fundamental events table (one row per event, one vehicle per API call).

---

### 3.4 `aggregation.py` — Fleet Analytics & Vehicle Benchmarking

Operates on the **master events table** (already populated by service.py). Does not fetch OBD data — it only reads and aggregates stored events.

#### Vehicle Flag Thresholds

| Flag | `clutch_riding_pct` range |
|---|---|
| **Good** | < 2% |
| **Normal** | 2% – 7% |
| **Bad** | 7% – 15% |
| **Critical** | > 15% |

#### `compute_fleet_summary(master_df) → DataFrame`

Groups by `client_name`. For each fleet:
- Counts unique vehicles
- Computes overall, clutch-riding, and normal-riding: distance, consumption, mileage
- Computes `fleet_clutch_riding_pct` = fleet-level clutch distance / total distance

#### `compute_vehicle_benchmarks(master_df) → DataFrame`

Groups by `client_name` + `vehicleNumber`. For each vehicle:
- Computes the same distance/consumption/mileage breakdown
- Computes `vehicle_clutch_riding_pct`
- Looks up `fleet_clutch_riding_pct` from fleet summary (reference benchmark)
- Applies the 4-tier flag using absolute thresholds
- Sorts by `client_name` ASC, `vehicle_clutch_riding_pct` DESC (worst vehicles first)

#### `run_fleet_aggregation(master_df) → dict`

Master entry point. Returns:
```python
{
  "fleet_summary":      [ { client_name, vehicle_count, ...metrics, fleet_clutch_riding_pct } ],
  "vehicle_benchmarks": [ { client_name, vehicleNumber, ...metrics, vehicle_flag } ],
  "flagged_vehicles": {
      "critical": [...],
      "bad":      [...],
      "normal":   [...],
      "good":     [...]
  }
}
```

---

## 4. Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    RAW OBD DATABASE                         │
│               (obd_data_history table)                      │
└──────────────────────┬──────────────────────────────────────┘
                       │  fetch_obd_for_clutch()
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                     service.py                              │
│                                                             │
│  raw OBD df                                                 │
│      │                                                      │
│      ├─ cycle segmentation (on-the-fly / stored / manual)   │
│      │                                                      │
│      └─ per cycle:                                          │
│            clean_and_flag_data()    ← logic.py              │
│            detect_clutch_events()   ← logic.py              │
│                                                             │
│      merge_events()                 ← logic.py              │
│      compute_metrics()              ← logic.py              │
│                                                             │
│  returns { events: [...], metrics: {...} }                  │
└──────────────────────┬──────────────────────────────────────┘
                       │  persisted by API caller
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              FUNDAMENTAL EVENTS TABLE                       │
│   (one row per event, per vehicle, per time window)         │
└──────────────────────┬──────────────────────────────────────┘
                       │  master_df passed to aggregation
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                  aggregation.py                             │
│                                                             │
│  compute_fleet_summary()   → fleet KPIs per client          │
│  compute_vehicle_benchmarks() → vehicle KPIs + 4-tier flag  │
│  run_fleet_aggregation()   → { fleet_summary,               │
│                                vehicle_benchmarks,          │
│                                flagged_vehicles }           │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Key Design Decisions

### Mileage excludes Short events
Events under 10 seconds produce unreliable fuel readings because the OBD sensor update rate may not capture the full fuel delta for very brief events. Mileage is therefore computed as `sum(distance) / sum(fuel)` over Medium, Long, and Very Long events only.

### Iterative event merging
A single-pass merge would miss cascades (e.g. two back-to-back short clutch bursts separated by a short normal gap). The loop continues until a full pass finds no mergeable triplets.

### Per-packet DQM flags attached to events
Rather than silently discarding events from poor-quality cycles, DQM flags (`dqm_fuel_error`, `dqm_dist_error`, etc.) are carried forward on every event row. This allows downstream analytics to filter or weight events by data quality without losing the events themselves.

### Fleet threshold vs 4-tier absolute thresholds
- `service.py` / `compute_metrics()` returns `clutch_riding_pct` as a raw number — no opinion on what is "good" or "bad".
- `aggregation.py` applies the 4-tier classification using fixed thresholds (Good < 2%, Normal 2–7%, Bad 7–15%, Critical > 15%). The fleet's own `fleet_clutch_riding_pct` is included as a reference column for fleet managers who want to see how a vehicle compares to its own fleet average.

### Separation of detection and aggregation
`logic.py` + `service.py` are called **per vehicle, per time window** by the API. `aggregation.py` is called once against the **entire stored dataset**. This split keeps the real-time API path fast and the analytics path independent.

---

## 6. Pending Work

| Item | File | Notes |
|---|---|---|
| GraphQL resolver | `resolver.py` | Wire `run_clutch_analysis` and `run_fleet_aggregation` to GraphQL queries |
| GraphQL schema | `schema.graphql` | Define `ClutchEvent`, `ClutchMetrics`, `VehicleBenchmark`, `FleetSummary` types |
| `__init__.py` | `__init__.py` | Export public functions from the package |
