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

#### 2.4 Driver Behavior
- Strong indicator of poor driving technique or fatigue
- Can correlate with other risky behaviors (harsh braking, over-speeding)

### Current Gap

No automated detection exists. Free-wheeling events go undetected and unreported across the fleet.

**Scope:** OBD-equipped vehicles only (`solution_type` filter applied).

---

## 3. Goals and Non-Goals

### Goals
- Detect free-wheeling events per vehicle per day using OBD telemetry
- Build **adaptive idle profiles per vehicle** (not global thresholds) so detection accounts for engine-to-engine variation
- Support fleet-wide reporting and driver coaching

### Non-Goals *(current phase)*
- Real-time alerting — batch offline detection only
- Non-OBD vehicles
- Writing results to a production database — outputs remain in DataFrames / CSV files

---

## 4. System Overview — Two-Phase Architecture

```
idling_history (DB)  ──┐
obd_data_history     ──┴──► [ Phase 1: Idle Profiler API ] ──► { uniqueid, rpm-low, rpm-high,
                                                                   engineload-low, engineload-high }

idle profiles (API output) ──┐
engineoncycles             ──┤──► [ Phase 2: Free Running Detector ] ──► free_running_df
obd_data_history           ──┘
```

---

### 4.1 Phase 1 — Idle Profiler API

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

### 4.2 Phase 2 — Free Running Detector

**Purpose:** Scan OBD telemetry within engine-on cycles and tag windows where the truck is moving at idle.

**Inputs:**
- `idle_profiles_df` — output of Phase 1
- `engineoncycles` (PostgreSQL, os_db) — engine-on cycle windows per vehicle
- `obd_data_history` (PostgreSQL, tracking_db) — raw OBD telemetry

**Logic:**
1. Fetch engine-on cycles for the detection date range
2. Drop cycles for vehicles without an idle profile
3. For each cycle, fetch OBD rows and run the state machine (below)
4. Tag detected event rows with a `free_running_id`
5. Flag events shorter than `MIN_EVENT_DURATION_S` with `flag_short_event = True` (not dropped)

**State machine — detection conditions:**

| Condition | Expression | Used in |
|-----------|-----------|---------|
| `rpm_ok` | `rpm` between `[idle_rpm_p25, idle_rpm_p75]` | Start only |
| `load_ok` | `engineload` between `[idle_engineload_p25, idle_engineload_p75]` | Start only |
| `acc_ok` | `accelerator_pedal_position ≤ idle_accelerator_p75` | Start + Continue |
| `speed_ok` | `vehiclespeed > MIN_SPEED_KMPH (5)` | Start + Continue |
| `start_cond` | `rpm_ok AND load_ok AND acc_ok AND speed_ok` | Event start |
| `continue_cond` | `acc_ok AND speed_ok` | Event continuation |

The event begins when `start_cond` is satisfied. It continues as long as `continue_cond` holds. Up to `NOISE_PATIENCE = 2` consecutive bad packets are tolerated before the event is ended. When an event closes, the end index is back-dated to the first bad packet (before the patience window).

**Event termination reasons:** `speed_zero`, `accelerator`, `end_of_data`

**Configuration parameters:**

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `MIN_EVENT_DURATION_S` | 10 | Events shorter than this are flagged (not dropped) |
| `MIN_SPEED_KMPH` | 5 | Speed below which the vehicle is treated as stopped |
| `NOISE_PATIENCE` | 2 | Consecutive bad packets tolerated before ending an event |
| `SAVE_EVERY` | 500 | Cycles between checkpoint saves (parquet batches) |

---

## 5. Data Sources

| Source | Type | Host:Port | Used In |
|--------|------|-----------|---------|
| `obd_data_history` | PostgreSQL table | tracking_db:5435 | Phase 1 + Phase 2 |
| `engineoncycles` | PostgreSQL table | os_db:5436 | Phase 2 |
| `idling_history` | PostgreSQL table | TBD | Phase 1 (API input) |

**Key OBD columns used:** `uniqueid`, `ts`, `rpm`, `engineload`, `vehiclespeed`, `accelerator_pedal_position`, `obddistance`, `fuel_consumption`

**Key engineoncycles columns used:** `cycle_id`, `uniqueid`, `cycle_start_ts`, `cycle_end_ts`, `cycle_date`

---

## 6. Output Artifacts

| Artifact | Format | Description |
|----------|--------|-------------|
| Phase 1 API response | JSON / DataFrame | Per-vehicle: `uniqueid`, `rpm-low`, `rpm-high`, `engineload-low`, `engineload-high` |
| `idle_profiles.csv` | CSV | Optional export of Phase 1 API output for inspection / audit |
| `free_running_df` | DataFrame | All OBD rows within detected events, tagged with `free_running_id` and `flag_short_event` |
| `free_running_detection.csv` | CSV | Export of `free_running_df` |
| `event_summary` | DataFrame | One row per event: `free_running_id`, vehicle, start/end timestamps, duration (s), distance (km), fuel consumed (L), average speed |
| `phase2_checkpoints/` | Parquet batches | Resume state — allows re-running without reprocessing completed cycles |

---

## 7. Algorithm Refinement Areas

These are known weaknesses to address before productionization.

| # | Issue | Description | Proposed Direction |
|---|-------|-------------|-------------------|
| 1 | **NULL handling** | If `idle_engineload_p25/p75` or `idle_accelerator_p75` is NaN in the profile, `load_ok` and `acc_ok` default to `True` (all rows pass). This silently disables part of the detection condition. | Define an explicit NULL handling policy: either exclude the vehicle entirely, fall back to fleet-average values, or flag affected events separately |
| 2 | **Fixed percentile bands** | P25–P75 was chosen heuristically. For vehicles with tightly-clustered idle signals the band may be too wide, accepting non-idle states; for noisy vehicles it may be too narrow, missing real idle. | Evaluate P10–P90 and adaptive width (e.g., IQR-based); measure impact on per-vehicle false positive rate |
| 3 | **NOISE_PATIENCE tuning** | Value of 2 is not empirically validated. Too low → events broken by single bad OBD packets; too high → non-free-wheeling intervals absorbed into events. | A/B test on a manually labeled sample; measure event fragmentation rate |
| 4 | **MIN_EVENT_DURATION_S** | 10 s threshold was not validated against real events. Short events may be GPS/OBD noise; very long events are the highest safety risk. | Determine threshold from labeled data; consider separate reporting tiers (warn vs. critical) |
| 5 | **Profile staleness** | Idle profiles are computed from the `idling_history` table but with no defined refresh cadence. A vehicle's idle characteristics may change (engine wear, recalibration). | Define a re-profiling schedule (weekly or monthly); the API caller is responsible for re-invoking with updated `idling_history` rows |
| 6 | **Cold-start problem** | Vehicles with fewer than 5 valid idling events receive no profile and are silently skipped in Phase 2. | Fall back to a fleet-average idle profile for cold-start vehicles; flag their detections separately |
| 7 | **Asymmetric start/continue conditions** | `start_cond` checks all four signals (RPM, load, accelerator, speed). `continue_cond` only checks accelerator and speed — RPM and engine load are not re-verified during an event. This can allow non-idle RPM/load segments to extend an event. | Evaluate tightening `continue_cond` to also verify RPM and load; measure effect on event duration and fragmentation |

---

## 8. Success Metrics

| Metric | Definition | Target |
|--------|------------|--------|
| **Profile coverage** | % of OBD-equipped vehicles with a valid idle profile built | > 80% of active vehicles |
| **Precision** | % of detected events confirmed as true free-wheeling on manual audit sample | TBD — requires labeled ground truth |
| **Recall** | % of actual free-wheeling events detected | TBD — no labeled dataset exists yet |
| **Event rate baseline** | Average detected events per vehicle per day | Establish from first production run |
| **Brake stress proxy** | Average event duration (s) × average speed (km/h) on detected events | Establish baseline; track over time |
| **Short-event rate** | % of events flagged as `flag_short_event = True` | Inform `MIN_EVENT_DURATION_S` tuning |

**Immediate next step for recall:** Identify a labeled dataset (dashcam footage + OBD sync, or driver logs) to enable precision/recall measurement.

---

## 9. Performance and Scalability

- **Parallel fetching:** `MAX_WORKERS = 8` concurrent DB connections for Phase 1 OBD fetching
- **Checkpoint-resume:** Phase 2 saves progress every `SAVE_EVERY = 500` cycles to `phase2_checkpoints/` as parquet batches; interrupted runs can resume without re-processing completed cycles
- **In-memory per cycle:** Each engine-on cycle is processed independently; the full dataset is never loaded into memory at once
- **Connection pools:** tracking_db uses a `ThreadedConnectionPool`; os_db uses a single read-only connection

---

## 10. Constraints and Limitations

| Constraint | Detail |
|------------|--------|
| OBD vehicles only | Detection requires `rpm`, `engineload`, `vehiclespeed`, `accelerator_pedal_position`; non-OBD vehicles are excluded |
| Batch only | No real-time detection; minimum latency is one full batch run after the detection window closes |
| DB credentials in notebook | Credentials are hardcoded — a security risk for any production deployment |
| No production DB write | Results are written only to local CSVs; integration into a reporting database is out of scope for this phase |
| Idling history window | Profile quality depends on the recency and volume of idling events in the CSV; stale or sparse data reduces detection reliability |

---

## 11. Open Questions and Decisions Needed

1. **`continue_cond` scope:** Should event continuation re-verify RPM and engine load, or is checking accelerator + speed sufficient? (See Refinement Area #7)
2. **Cold-start fallback:** What fleet-average idle values should be used for vehicles with fewer than 5 valid events? Who owns maintaining this fleet average?
3. **Re-profiling cadence:** Should idle profiles be rebuilt weekly, monthly, or event-triggered (e.g., after an engine replacement flag)?
4. **Short event policy:** Should events shorter than `MIN_EVENT_DURATION_S` be excluded from reports entirely, kept with a warning flag, or reported in a separate low-confidence tier?
5. **Labeled dataset:** What source of ground truth can be used to measure precision and recall? (Options: dashcam review, driver self-report, matched gear position data if available)
6. **Alert integration:** What is the timeline and owner for real-time alerting (out of scope for v1 but a likely follow-on requirement)?
7. **Alert integration:** What is the timeline and owner for real-time alerting (out of scope for v1 but a likely follow-on requirement)?

---

## 12. Verification Checklist

- [ ] All threshold values cross-checked against `free_running_detection.ipynb` (confirmed: `MIN_OBD_ROWS=3`, `MIN_VALID_EVENTS=5`, `MAX_IDLE_RPM=1000`, `MAX_IDLE_ENGINE_LOAD=35`, `MAX_IDLE_ACC_PEDAL=2`, `MIN_EVENT_DURATION_S=10`, `MIN_SPEED_KMPH=5`, `NOISE_PATIENCE=2`, `MAX_WORKERS=8`, `SAVE_EVERY=500`)
- [ ] Phase 1 API contract confirmed: input = `idling_history` table rows; output = `uniqueid, rpm-low, rpm-high, engineload-low, engineload-high`
- [ ] `idling_history` table schema and host/port confirmed with data engineering
- [ ] State machine logic (start vs. continue conditions, noise patience back-dating) reviewed against notebook code
- [ ] NULL handling behavior for `load_ok` / `acc_ok` confirmed against notebook (`np.ones` fallback)
- [ ] Engineering sign-off on Algorithm Refinement Areas (Section 7)
- [ ] Business / fleet operations sign-off on Success Metrics (Section 8)
- [ ] Security review: DB credential handling before any production deployment

---

*This PRD was generated from `free_running_detection.ipynb` (revision as of 2026-03-12). Any notebook changes should be reflected here before productionization.*
