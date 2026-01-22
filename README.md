# Clutch Riding Analysis System

## Overview

This project implements a comprehensive analysis system to detect and quantify clutch riding behavior in commercial vehicles. Clutch riding is a harmful driving practice where drivers keep the clutch pressed while the vehicle is moving at speed with the accelerator engaged, leading to excessive wear, fuel wastage, and increased operational costs.

## Table of Contents

- [What is Clutch Riding?](#what-is-clutch-riding)
- [Detection Criteria](#detection-criteria)
- [System Architecture](#system-architecture)
- [Data Flow](#data-flow)
- [Processing Logic](#processing-logic)
- [Output Structure](#output-structure)
- [Notebooks](#notebooks)
- [Data Quality Handling](#data-quality-handling)
- [Usage](#usage)

---

## What is Clutch Riding?

**Clutch riding** occurs when a driver keeps the clutch pedal pressed while:
- The vehicle is moving (speed > 20 km/h)
- The accelerator pedal is pressed (> 2%)
- The engine is running

This practice causes:
- **Premature clutch wear** - Significantly reduces clutch life
- **Fuel wastage** - Poor combustion efficiency
- **Increased maintenance costs** - More frequent clutch replacements
- **Safety concerns** - Reduced vehicle control

---

## Detection Criteria

An event is classified as **clutch riding** when ALL of the following conditions are met simultaneously:

```
clutch_switch_status == 'Pressed'
AND vehiclespeed > 20 km/h
AND accelerator_pedal_pos > 2%
```

### Additional Classification

- **Significant Consecutive Pattern**: Events with ≥5 consecutive packets (indicating sustained clutch riding)
- **Duration Categories**:
  - Short: < 10 seconds
  - Medium: 10-30 seconds
  - Long: 30-60 seconds
  - Very Long: > 60 seconds

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Data Sources                                  │
│  ┌──────────────────────┐    ┌──────────────────────┐          │
│  │  Engine Cycles DB    │    │  OBD Data History    │          │
│  │  (cycle metadata)    │    │  (telemetry packets) │          │
│  └──────────────────────┘    └──────────────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              Parallel Data Fetching (20 Workers)                 │
│  - Fetch OBD data for each engine cycle                         │
│  - ThreadPoolExecutor for concurrent processing                 │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                Event Detection & Processing                      │
│  - Apply detection criteria                                     │
│  - Group consecutive clutch riding packets                      │
│  - Calculate event metrics (duration, distance, fuel)          │
│  - Flag data quality issues                                     │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              Multi-Level Aggregation                            │
│  ┌───────────────┐  ┌───────────────┐  ┌──────────────────┐   │
│  │ Event-Level   │  │  Daily Stats  │  │ Vehicle Stats    │   │
│  │ (individual)  │  │  (aggregated) │  │ (monthly total)  │   │
│  └───────────────┘  └───────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Visualization & Reporting                       │
│  - Time-series analysis                                         │
│  - Vehicle performance rankings                                 │
│  - Executive dashboards                                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Data Flow

### 1. Input Data

#### Engine Cycles Data (`engineoncycles_*.csv`)
- **Purpose**: Defines the analysis scope
- **Columns**:
  - `cycle_id`: Unique identifier for each engine cycle
  - `uniqueid`: Vehicle identifier
  - `cycle_start_ts`: Cycle start timestamp (Unix)
  - `cycle_end_ts`: Cycle end timestamp (Unix)
  - `cycle_date`: Date in YYYY-MM-DD format
  - `cycle_duration_sec`: Cycle duration in seconds

#### OBD Data (from PostgreSQL database)
- **Table**: `obd_data_history`
- **Key Columns**:
  - `uniqueid`: Vehicle identifier
  - `ts`: Timestamp (Unix)
  - `vehiclespeed`: Speed in km/h
  - `rpm`: Engine RPM
  - `clutch_switch_status`: 'Pressed' or 'Released'
  - `accelerator_pedal_pos`: Accelerator position (0-100%)
  - `fuel_consumption`: Cumulative fuel consumption (liters)
  - `obddistance`: Cumulative distance (meters)

### 2. Processing Pipeline

#### Step 1: Data Fetching
```python
def fetch_obd_data_for_cycle(cycle_row, engine):
    """Fetch all OBD packets for a given cycle time window"""
    - Query OBD data between cycle_start_ts and cycle_end_ts
    - Return complete telemetry data for the cycle
```

#### Step 2: Event Detection
```python
def process_cycle_data(obd_df, cycle_id, cycle_date):
    """Detect clutch riding events within a cycle"""

    # 1. Data Cleaning
    - Convert columns to numeric
    - Handle NULL values with forward fill
    - Flag cycles with missing fuel data

    # 2. Calculate Event Metrics
    - event_distance = diff(obddistance)
    - event_fuel = diff(fuel_consumption)
    - event_duration = diff(ts)

    # 3. Apply Detection Logic
    is_clutch_riding = (
        (clutch_status == 'Pressed') &
        (speed > 20) &
        (accelerator > 2)
    )

    # 4. Group Consecutive Events
    - Identify continuous clutch riding periods
    - Aggregate metrics for each event

    # 5. Validate & Filter
    - Remove unrealistic events (distance > 20km, fuel > 5L, duration > 30min)
    - Skip events with < 2 packets

    return {
        'events': DataFrame of detected events,
        'processed_data': All packets with flags,
        'data_quality': Quality metrics
    }
```

#### Step 3: Aggregation
- **Event-Level**: Individual clutch riding instances
- **Cycle-Level**: Summary per engine cycle
- **Daily-Level**: Aggregated daily metrics
- **Vehicle-Level**: Monthly performance per vehicle
- **Fleet-Level**: Overall monthly statistics

---

## Output Structure

### 1. Event-Level Data (`clutch_riding_events_*.csv`)

Each row represents a single clutch riding event.

| Column | Type | Description |
|--------|------|-------------|
| `cycle_id` | String | Engine cycle identifier |
| `cycle_date` | Date | Date of the cycle (YYYY-MM-DD) |
| `uniqueid` | String | Vehicle identifier |
| `event_start_ts` | Integer | Event start timestamp (Unix) |
| `event_end_ts` | Integer | Event end timestamp (Unix) |
| `event_start_datetime` | Datetime | Human-readable start time |
| `event_end_datetime` | Datetime | Human-readable end time |
| `event_duration_sec` | Float | Duration in seconds |
| `event_distance_m` | Float | Distance traveled (meters) |
| `event_distance_km` | Float | Distance traveled (kilometers) |
| `event_fuel_liters` | Float | Fuel consumed (liters) |
| `event_mileage_kmpl` | Float | Mileage during event (km/L) |
| `avg_speed_kmh` | Float | Average speed during event |
| `avg_rpm` | Integer | Average engine RPM during event |
| `consecutive_packet_count` | Integer | Number of consecutive packets |
| `consecutive_duration_category` | String | Duration classification |
| `significant_consecutive_flag` | Boolean | True if ≥5 consecutive packets |
| `fuel_null_flag` | Boolean | True if fuel data was NULL |
| `event_index` | Integer | Event number within the cycle |

**Key Metrics**:
- **Typical Dataset Size**: 2.5M+ events for monthly data
- **File Format**: CSV (Excel not supported due to row limits)

### 2. Daily Statistics (`daily_statistics_*.csv`)

Aggregated metrics per day.

| Column | Type | Description |
|--------|------|-------------|
| `date` | Date | Analysis date |
| `total_cycles` | Integer | Total engine cycles on this day |
| `cycles_with_clutch_riding` | Integer | Cycles that had clutch riding events |
| `vehicles_with_clutch_riding` | Integer | Unique vehicles with clutch riding |
| `total_events` | Integer | Total clutch riding events detected |
| `total_duration_sec` | Float | Total duration (seconds) |
| `total_duration_hours` | Float | Total duration (hours) |
| `total_distance_km` | Float | Total distance during clutch riding |
| `total_fuel_liters` | Float | Total fuel wasted |
| `avg_speed_kmh` | Float | Average speed across all events |
| `avg_rpm` | Integer | Average RPM across all events |
| `significant_consecutive_events` | Integer | Events with ≥5 consecutive packets |
| `clutch_riding_cycle_pct` | Float | % of cycles with clutch riding |

**Use Cases**:
- Identify problematic days
- Track daily trends
- Week-over-week comparisons
- Day-of-week patterns

### 3. Vehicle Statistics (`vehicle_statistics_*.csv`)

Monthly performance summary per vehicle.

| Column | Type | Description |
|--------|------|-------------|
| `uniqueid` | String | Vehicle identifier |
| `total_cycles` | Integer | Total engine cycles for this vehicle |
| `cycles_with_clutch_riding` | Integer | Cycles with clutch riding events |
| `days_active` | Integer | Number of days vehicle was active |
| `total_events` | Integer | Total clutch riding events |
| `total_duration_sec` | Float | Total clutch riding duration (seconds) |
| `total_duration_hours` | Float | Total clutch riding duration (hours) |
| `total_distance_km` | Float | Total distance during clutch riding |
| `total_fuel_liters` | Float | Total fuel wasted |
| `avg_speed_kmh` | Float | Average speed during clutch riding |
| `avg_rpm` | Integer | Average RPM during clutch riding |
| `significant_consecutive_events` | Integer | Events with sustained clutch riding |
| `clutch_riding_cycle_pct` | Float | % of vehicle's cycles with clutch riding |
| `avg_events_per_day` | Float | Average events per active day |

**Sorting**: Ordered by `total_events` descending (worst offenders first)

**Use Cases**:
- Identify vehicles requiring driver training
- Rank fleet performance
- Target maintenance interventions

### 4. Data Quality Report (`data_quality_issues_*.xlsx`)

Flags cycles with data quality concerns.

| Column | Type | Description |
|--------|------|-------------|
| `cycle_id` | String | Cycle identifier |
| `cycle_date` | Date | Cycle date |
| `total_packets` | Integer | Total OBD packets in cycle |
| `fuel_all_null` | Boolean | All fuel data is NULL |
| `distance_all_null` | Boolean | All distance data is NULL |
| `speed_all_null` | Boolean | All speed data is NULL |
| `rpm_all_null` | Boolean | All RPM data is NULL |
| `clutch_all_null` | Boolean | All clutch status is NULL |
| `accelerator_all_null` | Boolean | All accelerator data is NULL |
| `has_critical_issues` | Boolean | Critical data missing (distance) |

---

## Notebooks

### 1. `clutch_riding_engineoncycle.ipynb`
**Purpose**: Original single-day analysis template

**Functionality**:
- Process single day of engine cycles
- Detect clutch riding events
- Generate event and cycle-level statistics
- Basic visualization

**Use Case**: Ad-hoc daily analysis

---

### 2. `clutch_riding_monthly_analysis.ipynb`
**Purpose**: Large-scale monthly processing

**Key Features**:
- Parallel processing with 20 workers
- Handles 180K+ cycles efficiently
- Multi-level aggregation (event, daily, vehicle, monthly)
- Automatic data quality flagging
- Smart file export (CSV for large datasets, Excel when possible)

**Configuration**:
```python
NUM_WORKERS = 20           # Parallel workers
BATCH_SIZE = 1000          # Memory management
OUTPUT_DIR = './outputs_monthly'
```

**Execution Time**: Several hours for 180K+ cycles

**Output**:
- `clutch_riding_events_monthly_*.csv` (2.5M+ rows)
- `daily_statistics_monthly_*.csv` (~30 rows)
- `vehicle_statistics_monthly_*.csv` (~1500 rows)
- `data_quality_issues_monthly_*.xlsx`

---

### 3. `clutch_riding_visualization_no_seaborn.ipynb`
**Purpose**: Single-day visualization dashboard

**Input Files**:
- `clutch_riding_events_YYYY-MM-DD.csv`
- `cycle_summary_YYYY-MM-DD.csv`

**Visualizations** (15+ charts):

1. **Event Duration Analysis**
   - Histogram of event durations
   - Duration category pie chart
   - Box plot by category

2. **Distance Analysis**
   - Event distance distribution
   - Cumulative distance over time

3. **Fuel Analysis**
   - Fuel consumption per event
   - Mileage during clutch riding
   - Fuel waste by duration category

4. **Speed & RPM Analysis**
   - Speed distribution during clutch riding
   - RPM patterns
   - Speed vs. RPM correlation

5. **Consecutive Pattern Analysis**
   - Packet count distribution
   - Significant vs. normal events

6. **Cycle-Level Comparisons**
   - Clutch riding cycles vs. normal cycles
   - Comparative statistics

7. **Correlation Heatmap**
   - Relationships between metrics

8. **Executive Summary Dashboard**
   - Multi-panel overview
   - Key metrics and trends

**Export**: `clutch_riding_summary_dashboard_*.xlsx`

**Technology**: Pure matplotlib (no seaborn dependency)

---

### 4. `clutch_riding_monthly_visualization.ipynb`
**Purpose**: Monthly trend analysis and vehicle rankings

**Auto-Detection**: Automatically finds latest files in `./outputs_monthly/`

**Visualizations**:

1. **Time-Series Analysis**
   - Daily event trends (line charts)
   - Daily duration and distance trends
   - Daily fuel waste trends

2. **Week-over-Week Analysis**
   - Weekly aggregations
   - Week comparison charts

3. **Day-of-Week Patterns**
   - Average events by weekday
   - Operational patterns

4. **Monthly Aggregations**
   - Overall monthly statistics
   - Month-to-month comparisons

5. **Vehicle Performance Rankings**
   - Top 20 offenders (by events)
   - Top 20 offenders (by fuel waste)
   - Top 20 offenders (by duration)

6. **Projected Annual Impact**
   - Extrapolated yearly metrics
   - Cost impact estimates (₹100/liter assumed)

**Export**: `clutch_riding_monthly_dashboard.xlsx`

---

## Data Quality Handling

### NULL Fuel Data Strategy

**Problem**: Some cycles have NULL fuel consumption data

**Solution**: Flag but don't exclude
```python
# Flag NULL fuel cycles
data_quality['fuel_all_null'] = df['fuel_consumption'].isna().all()

# Store flag in event data
event['fuel_null_flag'] = data_quality['fuel_all_null']

# Calculate mileage only for valid fuel data
if not fuel_null_flag and event_fuel > 0:
    event_mileage = distance / fuel
else:
    event_mileage = 0
```

**Benefits**:
- Retain all clutch riding events
- Track data quality per event
- Filter by fuel_null_flag when analyzing fuel metrics
- Transparent reporting

### Data Validation

**Distance Validation**:
```python
# Remove negative distances (sensor resets)
df.loc[df['event_distance'] < 0, 'event_distance'] = 0

# Cap unrealistic distances (> 5km per packet)
df.loc[df['event_distance'] > 5000, 'event_distance'] = 0
```

**Fuel Validation**:
```python
# Remove negative fuel (sensor issues)
df.loc[df['event_fuel'] < 0, 'event_fuel'] = 0

# Cap unrealistic fuel (> 2L per packet)
df.loc[df['event_fuel'] > 2, 'event_fuel'] = 0
```

**Event-Level Filtering**:
```python
# Skip unrealistic events
if event_distance_m > 20000:     # > 20km
    continue
if event_fuel_liters > 5:         # > 5L
    continue
if event_duration_sec > 1800:     # > 30 minutes
    continue
```

---

## Usage

### Single-Day Analysis

1. Prepare your daily cycle data
2. Open `clutch_riding_engineoncycle.ipynb`
3. Update the cycle file path
4. Run all cells
5. Review event-level and cycle-level outputs

### Monthly Analysis

1. Prepare monthly cycle file: `engineoncycles_*.csv`
2. Open `clutch_riding_monthly_analysis.ipynb`
3. Configure:
   ```python
   ENGINE_CYCLES_FILE = 'engineoncycles_YYYYMMDDHHMMSS.csv'
   NUM_WORKERS = 20  # Adjust based on server capacity
   ```
4. Run all cells (will take several hours)
5. Output saved to `./outputs_monthly/`

### Monthly Visualization

1. Ensure monthly analysis completed successfully
2. Open `clutch_riding_monthly_visualization.ipynb`
3. Run all cells (auto-detects latest files)
4. Review time-series trends and vehicle rankings
5. Export: `clutch_riding_monthly_dashboard.xlsx`

### Single-Day Visualization

1. Prepare event and cycle summary CSVs
2. Open `clutch_riding_visualization_no_seaborn.ipynb`
3. Update file paths:
   ```python
   events_file = 'clutch_riding_events_2025-12-25.csv'
   summary_file = 'cycle_summary_2025-12-25.csv'
   ```
4. Run all cells
5. Export: `clutch_riding_summary_dashboard_*.xlsx`

---

## Key Insights & Metrics

### Typical Monthly Results (Sample)

**Fleet Overview**:
- Total Vehicles: ~1,500
- Vehicles with Clutch Riding: ~1,200 (80%)
- Total Engine Cycles: ~187,000
- Cycles with Clutch Riding: ~60,000 (32%)

**Impact**:
- Total Events Detected: ~2.5 million
- Total Duration: ~8,000 hours
- Total Distance: ~120,000 km
- Total Fuel Wasted: ~5,000 liters
- Estimated Cost: ₹500,000 (@₹100/L)

**Daily Averages**:
- Events per Day: ~80,000
- Duration per Day: ~250 hours
- Fuel Wasted per Day: ~160 liters

**Vehicle Performance**:
- Avg Events per Vehicle: ~2,000
- Worst Offender: ~15,000 events/month
- Vehicles with >100 events: ~800

---

## Technical Requirements

### Dependencies

```python
pandas>=1.5.0
numpy>=1.23.0
matplotlib>=3.6.0
sqlalchemy>=2.0.0
psycopg2>=2.9.0
tqdm>=4.64.0
openpyxl>=3.0.0  # For Excel export
```

### Database

- **Host**: PostgreSQL
- **Required Tables**:
  - `obd_data_history` (tracking_db)
- **Access**: Read-only access sufficient

### Hardware Recommendations

**For Monthly Processing**:
- CPU: 8+ cores (for parallel processing)
- RAM: 16GB+ (for large DataFrames)
- Storage: 5GB+ free space (for output files)
- Network: Stable connection to database

---

## Performance Optimization

### Parallel Processing
```python
# Utilize ThreadPoolExecutor for I/O-bound operations
with ThreadPoolExecutor(max_workers=20) as executor:
    futures = {executor.submit(fetch_and_process_cycle, row): idx
               for idx, row in cycles_df.iterrows()}
```

### Memory Management
- Process in batches to avoid memory overflow
- Use chunked CSV reading for large files
- Clear intermediate DataFrames after aggregation

### Database Optimization
- Use indexed queries on `uniqueid` and `ts`
- Fetch only required columns
- Use connection pooling

---

## Future Enhancements

1. **Real-time Alerts**: Trigger notifications for excessive clutch riding
2. **Driver Scoring**: Individual driver behavior scores
3. **Predictive Maintenance**: Estimate clutch replacement schedules
4. **Route Analysis**: Correlate with GPS data to identify problematic routes
5. **Comparative Analytics**: Fleet benchmarking across regions
6. **Machine Learning**: Predict clutch failures based on riding patterns

---

## Support & Maintenance

### Common Issues

**Issue**: Excel export fails with "sheet too large"
- **Solution**: Automatically handled - CSV export used for large datasets

**Issue**: Database connection timeout
- **Solution**: Increase connection timeout or reduce NUM_WORKERS

**Issue**: Memory error during processing
- **Solution**: Reduce BATCH_SIZE or process in smaller date ranges

**Issue**: Seaborn not available
- **Solution**: Use `clutch_riding_visualization_no_seaborn.ipynb` (pure matplotlib)

---

## License

Internal use only.

---

## Contact

For questions or issues, contact the Data Science team.

---

**Last Updated**: January 2026
