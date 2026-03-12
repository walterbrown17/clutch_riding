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
- Real-time alerting — results are stored after each batch run, not streamed live

---

## 4. System Overview — Two-Phase Architecture

```
idling_history (DB)  ──┐
obd_data_history     ──┴──► [ Phase 1: Idle Profiler API ] ──► { uniqueid, rpm-low, rpm-high,
                                                                   engineload-low, engineload-high }

idle profiles (Phase 1 output) ──┐
engineoncycles                 ──┤──► [ Phase 2: Free Wheeling Detector API ] ──► { free_wheeling events }
obd_data_history               ──┘
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

### 4.2 Phase 2 — Free Wheeling Detector API

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
| `obd_data_history` | PostgreSQL table | tracking_db:5435 | Phase 1 + Phase 2 |
| `engineoncycles` | PostgreSQL table | os_db:5436 | Phase 2 |
| `idling_history` | PostgreSQL table | TBD | Phase 1 (API input) |

**Key OBD columns used:** `uniqueid`, `ts`, `rpm`, `engineload`, `vehiclespeed`, `accelerator_pedal_pos` (DB column name; aliased to `accelerator_pedal_position` in processing), `obddistance`, `fuel_consumption`

**Key engineoncycles columns used:** `cycle_id`, `uniqueid`, `cycle_start_ts`, `cycle_end_ts`, `cycle_date`

---

## 6. Output Artifacts

| Artifact | Phase | Stored To | Description |
|----------|-------|-----------|-------------|
| Phase 1 API response | 1 | `free-wheeling-profile` | Per-vehicle idle profile: `uniqueid`, `rpm-low`, `rpm-high`, `engineload-low`, `engineload-high` |
| Phase 2 API response | 2 | `free-wheeling-events` | Detected free-wheeling events: `free_wheeling_id`, `uniqueid`, `start_ts`, `end_ts`, `duration_s`, `distance_km`, `avg_speed_kmph`, `flag_short_event` |

---

## 7. Algorithm Refinement Areas

These are known weaknesses to address before productionization.

| # | Issue | Description | Proposed Direction |
|---|-------|-------------|-------------------|
| 1 | **NULL handling** | If `engineload-low`/`engineload-high` is NaN in the profile, `load_ok` defaults to `True` (all rows pass this condition). This silently disables the engine load check for that vehicle. Accelerator thresholds are now hardcoded so they are no longer affected. | Define an explicit NULL handling policy for engineload: either exclude the vehicle from detection, fall back to fleet-average values, or flag affected events separately |
| 2 | **Fixed percentile bands** | P25–P75 was chosen heuristically. For vehicles with tightly-clustered idle signals the band may be too wide, accepting non-idle states; for noisy vehicles it may be too narrow, missing real idle. | Evaluate P10–P90 and adaptive width (e.g., IQR-based); measure impact on per-vehicle false positive rate |
| 3 | **NOISE_PATIENCE tuning** | Value of 2 is not empirically validated. Too low → events broken by single bad OBD packets; too high → non-free-wheeling intervals absorbed into events. | A/B test on a manually labeled sample; measure event fragmentation rate |
| 4 | **MIN_EVENT_DURATION_S** | 10 s threshold was not validated against real events. Short events may be GPS/OBD noise; very long events are the highest safety risk. | Determine threshold from labeled data; consider separate reporting tiers (warn vs. critical) |
| 5 | **Profile staleness** | Idle profiles are computed from the `idling_history` table but with no defined refresh cadence. A vehicle's idle characteristics may change (engine wear, recalibration). | Define a re-profiling schedule (weekly or monthly); the API caller is responsible for re-invoking with updated `idling_history` rows |
| 6 | **Cold-start problem** | Vehicles with fewer than 5 valid idling events receive no profile and are silently skipped in Phase 2. | Fall back to a fleet-average idle profile for cold-start vehicles; flag their detections separately |
| 7 | **Asymmetric start/continue conditions** | `start_cond` requires `accelerator == 0`; `continue_cond` allows `accelerator <= 2`. RPM and engine load are checked only at event start — not re-verified during continuation. A rising RPM/load mid-event will not end the event. | Evaluate re-verifying RPM and load during continuation; measure impact on event duration and false positive rate |

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

- **Parallel fetching:** `MAX_WORKERS = 8` concurrent DB connections; Phase 2 parallelises at the vehicle level — one thread per vehicle
- **Batched OBD fetch:** Phase 2 issues one DB query per vehicle covering all its cycles, then uses binary search (`searchsorted`) to slice per cycle in memory — eliminates N round-trips per vehicle
- **In-memory per cycle:** State machine runs on each cycle slice independently; the full fleet dataset is never held in memory at once
- **Connection pools:** tracking_db uses a `ThreadedConnectionPool`; os_db uses a single read-only connection

---

## 10. Constraints and Limitations

| Constraint | Detail |
|------------|--------|
| OBD vehicles only | Detection requires `rpm`, `engineload`, `vehiclespeed`, `accelerator_pedal_position`; non-OBD vehicles are excluded |
| Batch only | No real-time detection; minimum latency is one full batch run after the detection window closes |
| DB credentials in API | Credentials must be managed via environment variables or a secrets manager before production deployment |
| Idling history window | Profile quality depends on the recency and volume of idling events in the `idling_history` table; stale or sparse data reduces detection reliability |

---

## 11. Open Questions and Decisions Needed

1. **`continue_cond` scope:** Should event continuation re-verify RPM and engine load, or is checking accelerator + speed sufficient? (See Refinement Area #7)
2. **Cold-start fallback:** What fleet-average idle values should be used for vehicles with fewer than 5 valid events? Who owns maintaining this fleet average?
3. **Re-profiling cadence:** Should idle profiles be rebuilt weekly, monthly, or event-triggered (e.g., after an engine replacement flag)?
4. **Short event policy:** Should events shorter than `MIN_EVENT_DURATION_S` be excluded from reports entirely, kept with a warning flag, or reported in a separate low-confidence tier?
5. **Labeled dataset:** What source of ground truth can be used to measure precision and recall? (Options: dashcam review, driver self-report, matched gear position data if available)
6. **Alert integration:** What is the timeline and owner for real-time alerting (out of scope for v1 but a likely follow-on requirement)?

---

## 12. Verification Checklist

- [ ] All threshold values cross-checked against `free_running_detection.ipynb` (confirmed: `MIN_OBD_ROWS=3`, `MIN_VALID_EVENTS=5`, `MAX_IDLE_RPM=1000`, `MAX_IDLE_ENGINE_LOAD=35`, `MAX_IDLE_ACC_PEDAL=2`, `MIN_EVENT_DURATION_S=10`, `MIN_SPEED_KMPH=5`, `NOISE_PATIENCE=2`, `MAX_WORKERS=8`, `SAVE_EVERY=500`)
- [ ] Phase 1 API contract confirmed: input = `idling_history` table rows; output = `uniqueid, rpm-low, rpm-high, engineload-low, engineload-high`
- [ ] Phase 2 API contract confirmed: input = Phase 1 profiles + date range; output = free-wheeling event records
- [ ] `idling_history` table schema and host/port confirmed with data engineering
- [ ] State machine logic (start vs. continue conditions, noise patience back-dating) reviewed against notebook code
- [ ] NULL handling behavior for `load_ok` confirmed (`np.ones` fallback when engineload profile is NaN); accelerator thresholds are now hardcoded so no longer affected
- [ ] Engineering sign-off on Algorithm Refinement Areas (Section 7)
- [ ] Business / fleet operations sign-off on Success Metrics (Section 8)
- [ ] Security review: DB credential handling before any production deployment

---

*This PRD was generated from `free_running_detection.ipynb` (revision as of 2026-03-12). Any notebook changes should be reflected here before productionization.*
