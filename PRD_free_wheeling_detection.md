## 1. Executive Summary

Heavy commercial trucks that coast in neutral while moving — "free-wheeling" — create safety hazards, accelerate mechanical wear, violate regulations, and signal poor driver technique. Yet today, no automated system detects or reports these events across the fleet.

This document describes a two-phase batch detection system that builds a per-vehicle idle fingerprint and then flags OBD telemetry windows where the truck is moving but the engine is only idling. All detection runs offline; outputs are DataFrames and CSV files available for fleet reporting and further analysis.

**Target users:** Fleet operators (reporting & compliance), data/engineering teams (refinement & productionization).

---

## 2. Problem Statement

### What is Free-Wheeling?

Free-wheeling occurs when a truck is physically moving but the engine is only idling — not producing drive torque. The engine is effectively disconnected from the drivetrain while the vehicle coasts (typically downhill or when decelerating).

### Why it Matters

#### 2.1 Safety *(most critical for heavy vehicles)*
- **Loss of engine braking** — on downhill gradients the driver relies entirely on friction brakes
- **Brake overheating / fade** — prolonged friction-only braking causes brake fade, significantly increasing stopping distance
- **Reduced vehicle control** — lower stability on curves or wet/slippery roads without engine drag
- **Longer stopping distances** — directly increases accident risk

#### 2.2 Mechanical Wear
- **Accelerated brake pad & disc wear** — over-reliance on friction brakes instead of engine braking
- **Turbocharger stress** — sudden re-engagement after free-wheeling can cause oil starvation or surge
- **Transmission stress** — abrupt clutch engagement after coasting puts shock load on the gearbox

#### 2.3 Regulatory / Compliance
- Free-wheeling in neutral is **illegal for heavy commercial vehicles** in several jurisdictions, including India under CMVR rules
- Violates fleet safety SOPs and OEM operating guidelines

---

## 3. Goals 

### Goals
- Detect free-wheeling events per vehicle per day using OBD telemetry
- Build **adaptive idle profiles per vehicle** (not global thresholds) so detection accounts for engine-to-engine variation
- Support fleet-wide reporting and driver coaching


---

## 4. System Overview — Two-Step Architecture

```
idling_history (DB)  ──┐
obd_data_history     ──┴──► [ Phase 1: Idle Profiler API ] ──► { uniqueid, rpm-low, rpm-high,
                                                                   engineload-low, engineload-high }

idle profiles (Phase 1 output) ──┐
engineoncycles                 ──┤──► [ Phase 2: Free Wheeling Detector API ] ──► { free_wheeling events }
obd_data_history               ──┘
```

---

### 4.1 Step 1 — Idle Profiler API

**Purpose:** Expose a service that accepts idling event rows for a set of vehicles, computes per-vehicle idle RPM and engine load bands from OBD telemetry, and returns a profile per vehicle.

**API contract:**

| | Detail |
|--|--------|
| **Input** | Rows from the `idling_history` table (one row per known idling event: `uniqueid`, `starttime`, `endtime`) |
| **Output** | One record per vehicle: `uniqueid`, `rpm-low`, `rpm-high`, `engineload-low`, `engineload-high` |

**Internal logic:**
1. Receive idling event rows from the `idling_history` table
2. For each vehicle, fetch OBD rows from `obd_data_history` that fall within each idling window
4. Discard events with fewer than `MIN_OBD_ROWS = 3` OBD rows
5. Discard events where the average signal exceeds any threshold:
   - `avg_rpm > MAX_IDLE_RPM (1000)`
   - `avg_engineload > MAX_IDLE_ENGINE_LOAD (35)`
   - `avg_accelerator > MAX_IDLE_ACC_PEDAL (2)`
6. If the vehicle has at least `MIN_VALID_EVENTS = 5` clean events, compute percentile bands:
   - `rpm-low` = `idle_rpm_p25`, `rpm-high` = `idle_rpm_p75`
   - `engineload-low` = `idle_engineload_p25`, `engineload-high` = `idle_engineload_p75`
7. Vehicles with fewer than `MIN_VALID_EVENTS` valid events are excluded from the output (cold-start problem — see Refinement Area #6)

**Configuration parameters:**

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `MIN_OBD_ROWS` | 3 | Minimum OBD rows per idling event to be usable |
| `MIN_VALID_EVENTS` | 5 | Minimum clean events needed to build a profile |
| `MAX_IDLE_RPM` | 1000 | RPM ceiling for a valid idling event |
| `MAX_IDLE_ENGINE_LOAD` | 35 | Engine load ceiling (%) for a valid idling event |
| `MAX_IDLE_ACC_PEDAL` | 2 | Accelerator pedal ceiling (%) for a valid idling event (used for filtering only, not in output) |
| `MAX_WORKERS` | 8 | Parallel DB threads for OBD fetching |

---

### 4.2 Step 2 — Free Wheeling Detector API

**Purpose:** Expose a service that accepts idle profiles (from Phase 1) and a detection date range, fetches engine-on cycles and OBD telemetry directly from the database, and returns detected free-wheeling events per vehicle.

**API contract:**

| | Detail |
|--|--------|
| **Input** | Idle profiles from Phase 1 (`uniqueid`, `rpm-low`, `rpm-high`, `engineload-low`, `engineload-high`) + detection date range (`start_date`, `end_date`) |
| **Output** | Detected free-wheeling events: `free_wheeling_id`, `uniqueid`, `start_ts`, `end_ts`, `duration_s`, `distance_km`, `avg_speed_kmph`, `flag_short_event` |

**Internal logic:**
1. Fetch engine-on cycles for the detection date range from `engineoncycles` (DB)
2. Drop cycles for vehicles not present in the provided idle profiles
3. For each vehicle, fetch **all OBD rows across all its cycles in a single query**, then slice per cycle using binary search (`searchsorted`) — avoids one DB round-trip per cycle
4. For each cycle slice, run the state machine (below)
5. Tag detected event rows with a `free_wheeling_id` — only rows where `accelerator_pedal_position <= 2` are tagged (rows where accelerator exceeded the threshold are excluded from the event boundary)
6. Flag events shorter than `MIN_EVENT_DURATION_S` with `flag_short_event = True` (not dropped)

**State machine — detection conditions:**

| Condition | Expression | Threshold | Used in |
|-----------|-----------|-----------|---------|
| `rpm_ok` | `rpm` between `[rpm-low, rpm-high]` | from Phase 1 profile | Start only |
| `load_ok` | `engineload` between `[engineload-low, engineload-high]` | from Phase 1 profile | Start only |
| `acc_start` | `accelerator_pedal_position == 0` | hardcoded | Start only |
| `acc_cont` | `accelerator_pedal_position <= 2` | hardcoded | Continue only |
| `speed_ok` | `vehiclespeed > MIN_SPEED_KMPH (5)` | hardcoded | Start + Continue |
| `start_cond` | `rpm_ok AND load_ok AND acc_start AND speed_ok` | — | Event start |

**State machine — event continuation and termination:**

| Situation | Behaviour |
|-----------|-----------|
| `acc_cont` fails (`accelerator > 2`) | Immediate event end; the failing row is excluded from the event |
| `speed_ok` fails | Patience counter incremented; event ends after `NOISE_PATIENCE = 2` consecutive failures; end index back-dated to first bad packet |
| Both conditions pass | Patience counter reset to 0 |
| End of cycle data | Event closed at last row |

**Event termination reasons:** `accelerator` (immediate), `speed_zero` (after patience), `end_of_data`

**Configuration parameters:**

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `MIN_EVENT_DURATION_S` | 10 | Events shorter than this are flagged (not dropped) |
| `MIN_SPEED_KMPH` | 5 | Speed below which the vehicle is treated as stopped |
| `NOISE_PATIENCE` | 2 | Consecutive speed-fail packets tolerated before ending an event (does not apply to accelerator failures) |

---

## 5. Data Sources

| Source | Type | Host:Port | Used In |
|--------|------|-----------|---------|
| `obd_data_history` | PostgreSQL table | tracking_db | Step 1 + Step 2 |
| `engineoncycles` | PostgreSQL table | os_db | Step 2 |
| `idling_history` | PostgreSQL table | tracking_db | Step 1 (API input) |

**Key OBD columns used:** `uniqueid`, `ts`, `rpm`, `engineload`, `vehiclespeed`, `accelerator_pedal_pos` , `obddistance`, `fuel_consumption`

**Key engineoncycles columns used:** `cycle_id`, `uniqueid`, `cycle_start_ts`, `cycle_end_ts`, `cycle_date`

---

## 6. Output Artifacts

| Artifact | Phase | Stored To | Description |
|----------|-------|-----------|-------------|
| Step 1 API response | 1 | `free-wheeling-profile` | Per-vehicle idle profile: `uniqueid`, `rpm-low`, `rpm-high`, `engineload-low`, `engineload-high` |
| Step 2 API response | 2 | `free-wheeling-events` | Detected free-wheeling events: `free_wheeling_id`, `uniqueid`, `start_ts`, `end_ts`, `duration_s`, `distance_km`, `avg_speed_kmph`, `flag_short_event` |
