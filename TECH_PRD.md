# Tech PRD — Clutch Riding API


## 1. Purpose

Detects and quantifies *clutch riding* behaviour from raw OBD telemetry per engine cycle.
Clutch riding is defined as: **clutch pedal held pressed while the vehicle is moving above 20 km/h with the accelerator also engaged** — a driving habit that causes premature clutch wear and degrades fuel efficiency.

The API produces two levels of output:

| Level | Description |
|---|---|
| **Event level** | Each contiguous segment of clutch-riding or normal-riding within a cycle |
| **Metrics level** | Aggregated KPIs — distance, fuel, mileage, and clutch% per vehicle/fleet |

---

## 2. Module Architecture

```
clutch_riding/
├── config.py        — Detection thresholds and duration constants
├── logic.py         — Core pipeline: DQM → detection → merging → metrics
├── service.py       — [stub] Orchestrates DB fetch + logic pipeline per cycle
├── resolver.py      — [stub] GraphQL resolver binding
├── aggregation.py   — [stub] Fleet/vehicle-level rollup from fundamental table
└── schema.graphql   — [stub] GraphQL type and query definitions
```

---

## 3. Processing Pipeline

```
Raw OBD packets (from DB)
        │
        ▼
┌─────────────────────────┐
│  1. clean_and_flag_data  │  ← Sensor checks, DQM flags, column normalization
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│  2. detect_clutch_events │  ← Packet-level classification → event grouping
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│  3. merge_events         │  ← Noise reduction (deduplication of boundary artefacts)
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│  4. compute_metrics      │  ← Aggregate KPIs (distance / fuel / mileage / %)
└─────────────────────────┘
```

### Step 1 — DQM Cleaning (`clean_and_flag_data`)

Produces quality flags for the cycle and normalises numeric columns.

- Cumulative sensors (`obddistance`, `fuel_consumption`) are **forward-filled** to handle mid-cycle dropouts.
- Missing columns are injected with `0` to prevent downstream KeyErrors.
- Speeds are coerced; impossibly high values (`>= 250 km/h`) are flagged.
- Time-gap continuity is checked; gaps `> 60 s` are flagged.

### Step 2 — Event Detection (`detect_clutch_events`)

Operates on packet-level diffs:

```
event_duration          = diff(ts)
event_distance_m        = diff(obddistance)
event_fuel_liters       = diff(fuel_consumption)
event_fuel_from_rate    = fuel_rate (L/h) × 5s / 3600
```

Haversine GPS distance is computed alongside OBD odometer as a cross-check.

**Clutch riding condition per packet:**
```
clutch_switch_status == 'Pressed'
AND vehiclespeed > 20 km/h
AND accelerator_pedal_pos > 2
```

Consecutive packets with the same state are grouped into a single event.
Events shorter than 2 seconds are discarded as noise.

### Step 3 — Event Merging (`merge_events`)

Iteratively resolves two boundary noise patterns:

| Pattern | Action |
|---|---|
| `normal(≥30s)` → `clutch(≤10s)` → `normal(≥30s)` | Merge all three → `normal_riding` |
| `clutch` → `normal(≤10s)` → `clutch` | Merge all three → `clutch_riding` |

Loop continues until no further merges are possible (handles cascades).

**Merge arithmetic:**
- `duration`, `distance_km`, `fuel_consumed_liters` → simple sum
- `avg_speed_kmh`, `avg_rpm` → duration-weighted average
- `severity` → re-derived from merged total duration
- `mileage_kmpl` → recomputed from merged totals

### Step 4 — Metrics Computation (`compute_metrics`)

Produces three segments: `overall`, `clutch_riding`, `normal_riding`.

**Mileage rule:**
```
mileage_kmpl = sum(distance_km) / sum(fuel_consumed_liters)
               using ONLY events where severity ∈ {Medium, Long, Very Long}
               (Short events excluded — too brief for accurate fuel reading)
```

**Clutch riding %:**
```
clutch_riding_pct = (clutch_distance_km / overall_distance_km) × 100
```
Distance-based, not fuel-based.

---

## 4. API Usage

### GraphQL Endpoint

```
POST /graphql
```

### Intended Query (schema to be defined in `schema.graphql`)

```graphql
query ClutchRidingAnalysis(
  $unique_id: String!
  $fromTime: Float!
  $toTime: Float!
) {
  clutchRidingAnalysis(
    unique_id: $unique_id
    fromTime: $fromTime
    toTime: $toTime
  ) {
    cycles {
      cycle_id
      cycle_start_ts
      cycle_end_ts
      cycle_duration_sec

      dqm {
        fuel_missing
        dist_missing
        rpm_missing
        clutch_missing
        large_time_gap_exists
        impossible_speed_detected
      }

      metrics {
        overall_distance_km
        overall_consumption_liters
        overall_mileage_kmpl
        clutch_riding_distance_km
        clutch_riding_consumption_liters
        clutch_riding_mileage_kmpl
        normal_riding_distance_km
        normal_riding_consumption_liters
        normal_riding_mileage_kmpl
        clutch_riding_pct
      }

      events {
        event_type
        event_start_ts
        event_end_ts
        duration_sec
        distance_km
        avg_speed_kmh
        avg_rpm
        severity
        fuel_consumed_liters
        mileage_kmpl
        dqm_fuel_error
        dqm_dist_error
        dqm_speed_error
        dqm_gap_error
      }
    }
  }
}
```

### Input Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `unique_id` | `String` | Yes | Vehicle device identifier (OBD unit ID) |
| `fromTime` | `Float` | Yes | Window start — UNIX epoch seconds |
| `toTime` | `Float` | Yes | Window end — UNIX epoch seconds |

---

## 5. Output Structure

The response is a **list of engine cycles**. Each cycle object contains three nested sections: `dqm`, `metrics`, and `events`. This mirrors the BMS API pattern where each engine cycle carries its own aggregated summary alongside its constituent detail records.

```
clutchRidingAnalysis
└── cycles[]                      ← one object per engine ON cycle
    ├── cycle_id
    ├── cycle_start_ts
    ├── cycle_end_ts
    ├── cycle_duration_sec
    ├── dqm { ... }               ← data quality flags for this cycle
    ├── metrics { ... }           ← aggregated KPIs for this cycle
    └── events[]                  ← individual clutch/normal segments within this cycle
```

### 5.1 Cycle-Level Fields

| Field | Type | Description |
|---|---|---|
| `cycle_id` | String | Unique identifier for the engine ON cycle |
| `cycle_start_ts` | Int | UNIX timestamp (seconds) — engine ON |
| `cycle_end_ts` | Int | UNIX timestamp (seconds) — engine OFF |
| `cycle_duration_sec` | Int | Total duration of the cycle in seconds |

---

### 5.2 DQM Flags (`dqm`)

Data quality metadata produced during `clean_and_flag_data()`. All flags are cycle-scoped — they reflect the quality of the raw OBD stream for that entire cycle and are propagated onto every event inside it.

| Field | Type | Description |
|---|---|---|
| `fuel_missing` | Boolean | `fuel_consumption` column absent or entirely NaN |
| `dist_missing` | Boolean | `obddistance` column absent or entirely NaN |
| `rpm_missing` | Boolean | `rpm` absent, all NaN, or all zero |
| `clutch_missing` | Boolean | `clutch_switch_status` absent or entirely NaN — clutch detection not possible for this cycle |
| `large_time_gap_exists` | Boolean | Any time gap `> 60 s` between consecutive packets |
| `impossible_speed_detected` | Boolean | Any packet with `vehiclespeed ≥ 250 km/h` |

---

### 5.3 Cycle Metrics (`metrics`)

Aggregated KPIs computed by `compute_metrics()` called on this cycle's merged event list. Three segments are reported: overall, clutch riding only, and normal riding only.

| Field | Type | Description |
|---|---|---|
| `overall_distance_km` | Float | Total distance across all events in this cycle (km) |
| `overall_consumption_liters` | Float | Total fuel consumed across all events in this cycle (L) |
| `overall_mileage_kmpl` | Float | Cycle mileage: sum_dist / sum_cons using Medium+ events only (km/L) |
| `clutch_riding_distance_km` | Float | Distance covered in clutch_riding events (km) |
| `clutch_riding_consumption_liters` | Float | Fuel consumed in clutch_riding events (L) |
| `clutch_riding_mileage_kmpl` | Float | Mileage for the clutch_riding segment only (km/L) |
| `normal_riding_distance_km` | Float | Distance covered in normal_riding events (km) |
| `normal_riding_consumption_liters` | Float | Fuel consumed in normal_riding events (L) |
| `normal_riding_mileage_kmpl` | Float | Mileage for the normal_riding segment only (km/L) |
| `clutch_riding_pct` | Float | `(clutch_distance / overall_distance) × 100` — primary KPI for this cycle |

> **Mileage rule:** `sum(distance_km) / sum(fuel_consumed_liters)` using only events where `severity ∈ {Medium, Long, Very Long}`. Short events (`< 10 s`) are excluded from both numerator and denominator.

---

### 5.4 Events List (`events[]`)

One object per contiguous clutch-riding or normal-riding segment detected within the cycle, after noise merging. Ordered by `event_start_epoch`.

| Field | Type | Description |
|---|---|---|
| `event_type` | String | `clutch_riding` or `normal_riding` |
| `event_start_ts` | Int | UNIX timestamp (seconds) — start of the segment |
| `event_end_ts` | Int | UNIX timestamp (seconds) — end of the segment |
| `duration_sec` | Float | Duration of the segment in seconds |
| `distance_km` | Float | Distance covered during the segment (km, from OBD odometer) |
| `avg_speed_kmh` | Float | Average vehicle speed during the segment (km/h) |
| `avg_rpm` | Float | Average engine RPM during the segment |
| `severity` | String | Duration tier — see table below |
| `fuel_consumed_liters` | Float | Fuel consumed during the segment (litres) |
| `mileage_kmpl` | Float | Segment mileage (km/L); `0.0` when fuel data unavailable |
| `dqm_fuel_error` | Boolean | Propagated from cycle DQM — fuel sensor issue |
| `dqm_dist_error` | Boolean | Propagated from cycle DQM — odometer issue |
| `dqm_speed_error` | Boolean | Propagated from cycle DQM — impossible speed detected |
| `dqm_gap_error` | Boolean | Propagated from cycle DQM — large time gap present |

#### Severity Categories

| Severity | Duration Range | Included in Mileage Calc? |
|---|---|---|
| `Short` | `< 10 s` | No |
| `Medium` | `10 s – 30 s` | Yes |
| `Long` | `30 s – 60 s` | Yes |
| `Very Long` | `≥ 60 s` | Yes |

---

## 6. Fundamental Tables

These two tables form the persistent storage layer for the clutch riding pipeline. `aggregation.py` reads from them to build fleet and vehicle-level reports.

---

### Table 1 — `clutch_riding_cycles`

One row per engine ON cycle. Stores the cycle envelope and all aggregated KPIs produced by `compute_metrics()` for that cycle.

**Primary key:** `(unique_id, cycle_id)` — composite natural key. `unique_id` alone does not uniquely identify a cycle (one vehicle has many cycles); `cycle_id` alone is already unique but pairing it with `unique_id` makes vehicle-scoped queries efficient without a join.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `unique_id` | VARCHAR | PK, NOT NULL | Vehicle device identifier (OBD unit ID) |
| `cycle_id` | VARCHAR | PK, NOT NULL | Unique engine cycle identifier |
| `cycle_start_ts` | BIGINT | NOT NULL | Cycle start — UNIX epoch seconds (engine ON) |
| `cycle_end_ts` | BIGINT | NOT NULL | Cycle end — UNIX epoch seconds (engine OFF) |
| `cycle_date` | DATE | NOT NULL | Calendar date of `cycle_start_ts` (UTC) — enables efficient day-level aggregation without epoch arithmetic |
| `cycle_duration_sec` | INT | NOT NULL | `cycle_end_ts - cycle_start_ts` |
| `overall_distance_km` | FLOAT | NOT NULL | Total distance across all events in this cycle (km) |
| `overall_consumption_liters` | FLOAT | NOT NULL | Total fuel consumed across all events in this cycle (L) |
| `overall_mileage_kmpl` | FLOAT | NOT NULL | Cycle mileage — `sum_dist / sum_cons` using Medium+ events only (km/L) |
| `clutch_riding_distance_km` | FLOAT | NOT NULL | Distance covered in `clutch_riding` events (km) |
| `clutch_riding_consumption_liters` | FLOAT | NOT NULL | Fuel consumed in `clutch_riding` events (L) |
| `clutch_riding_mileage_kmpl` | FLOAT | NOT NULL | Mileage for the clutch_riding segment only (km/L) |
| `normal_riding_distance_km` | FLOAT | NOT NULL | Distance covered in `normal_riding` events (km) |
| `normal_riding_consumption_liters` | FLOAT | NOT NULL | Fuel consumed in `normal_riding` events (L) |
| `normal_riding_mileage_kmpl` | FLOAT | NOT NULL | Mileage for the normal_riding segment only (km/L) |
| `clutch_riding_pct` | FLOAT | NOT NULL | `(clutch_distance / overall_distance) × 100` — primary KPI |
| `created_at` | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT now() | Row insertion time — used for CDC and audit |
| `updated_at` | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT now() | Last update time — updated via trigger on any column change; used for CDC |

> **CDC note:** `updated_at` should be maintained by a `BEFORE UPDATE` trigger (e.g. `SET updated_at = now()`). Do not rely on the application layer to set it — trigger-based updates are consistent and cannot be bypassed.

> **`cycle_duration_sec`** is a derived column (`cycle_end_ts - cycle_start_ts`). It is stored explicitly for query convenience so aggregation queries do not need to recompute it every time.

> **`overall_mileage_kmpl`, `clutch_riding_mileage_kmpl`, `normal_riding_mileage_kmpl`** are also derived (distance / consumption). Stored for the same reason — avoids recomputation in fleet aggregation queries that scan large date ranges.

---

### Table 2 — `events_for_clutch_riding`

One row per detected clutch-riding or normal-riding segment (after noise merging). Child table of `clutch_riding_cycles` via `(unique_id, cycle_id)`.

**Primary key:** `event_id` — surrogate natural key generated deterministically from the cycle and event properties (see generation rule below).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `event_id` | VARCHAR | PK, NOT NULL | Globally unique event identifier — see generation rule below |
| `unique_id` | VARCHAR | NOT NULL, FK | Vehicle device identifier — FK → `clutch_riding_cycles.unique_id` |
| `cycle_id` | VARCHAR | NOT NULL, FK | FK → `clutch_riding_cycles.cycle_id` |
| `event_type` | VARCHAR | NOT NULL | `clutch_riding` or `normal_riding` |
| `event_start_ts` | BIGINT | NOT NULL | UNIX timestamp (seconds) — start of the segment |
| `event_end_ts` | BIGINT | NOT NULL | UNIX timestamp (seconds) — end of the segment |
| `duration_sec` | FLOAT | NOT NULL | Duration of the segment in seconds |
| `distance_km` | FLOAT | NOT NULL | Distance covered during the segment (km) |
| `avg_speed_kmh` | FLOAT | NOT NULL | Average vehicle speed during the segment (km/h) |
| `avg_rpm` | FLOAT | NOT NULL | Average engine RPM during the segment |
| `severity` | VARCHAR | NOT NULL | Duration tier: `Short` / `Medium` / `Long` / `Very Long` |
| `fuel_consumed_liters` | FLOAT | NOT NULL | Fuel consumed during the segment (L) |
| `dqm_fuel_error` | BOOLEAN | NOT NULL | Propagated from cycle DQM — fuel sensor missing or fully null |
| `dqm_dist_error` | BOOLEAN | NOT NULL | Propagated from cycle DQM — OBD odometer missing or fully null |
| `dqm_speed_error` | BOOLEAN | NOT NULL | Propagated from cycle DQM — impossible speed detected (`≥ 250 km/h`) |
| `dqm_gap_error` | BOOLEAN | NOT NULL | Propagated from cycle DQM — large time gap (`> 60 s`) in raw data |
| `created_at` | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT now() | Row insertion time — used for CDC and audit |
| `updated_at` | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT now() | Last update time — maintained by `BEFORE UPDATE` trigger; used for CDC |

> **`mileage_kmpl` removed:** Previously listed as a stored column. It is a derived value (`distance_km / fuel_consumed_liters`) that can be computed on the fly. Storing it risks the stored value drifting out of sync with the source columns if either is updated. Consumers that need mileage per event should compute it at query time.

The `event_id` generation strategy is left to the tech team's discretion. It must guarantee global uniqueness across all events in the table.

---
