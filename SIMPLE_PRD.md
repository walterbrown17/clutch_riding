# Clutch Riding Analysis - Technical PRD
**Version:** 1.0
**Date:** January 19, 2026
**Purpose:** Generate day-wise clutch riding statistics per vehicle

---

## 1. Objective

Calculate daily clutch riding metrics (distance, fuel, mileage) for each vehicle and store in a final summary table for analysis and savings calculation.

**Hypothesis:** Clutch riding reduces fuel efficiency. By quantifying the impact, we can identify high-waste vehicles and calculate potential savings.

---

## 2. Data Flow

```
PostgreSQL (obd_data_history)
    ↓
Fetch per vehicle, per day
    ↓
Forward fill (obddistance, fuel_consumption)
    ↓
Create event-based features (next - current)
    ↓
Detect clutch riding events
    ↓
Calculate statistics
    ↓
Final Output Table
```

---

## 3. Input Table

### Source: `public.obd_data_history`

| Column | Type | Description | Unit |
|--------|------|-------------|------|
| `uniqueid` | VARCHAR | Vehicle ID | - |
| `ts` | INTEGER | Unix timestamp | seconds |
| `vehiclespeed` | NUMERIC | Vehicle speed | km/h |
| `rpm` | NUMERIC | Engine RPM | rev/min |
| `clutch_switch_status` | VARCHAR | Clutch status | Pressed/Released |
| `accelerator_pedal_pos` | NUMERIC | Accelerator position | % |
| `fuel_consumption` | NUMERIC | Cumulative fuel | liters |
| `obddistance` | NUMERIC | Cumulative odometer | meters |

**Query Strategy:** Per-vehicle queries for each day
```sql
SELECT * FROM obd_data_history
WHERE ts >= :day_start AND ts < :day_end
  AND uniqueid = :vehicle_id
ORDER BY ts ASC
```

---

## 4. Processing Steps

### Step 1: Forward Fill
Handle NULL values when vehicle is stationary
```python
df['obddistance'] = df['obddistance'].ffill()
df['fuel_consumption'] = df['fuel_consumption'].ffill()
```

### Step 2: Event-Based Feature Engineering
Calculate changes between consecutive timestamps:
```python
event_distance = next_obddistance - current_obddistance  # meters
event_fuel = next_fuel_consumption - current_fuel_consumption  # liters
event_duration = next_ts - current_ts  # seconds
```

**Validations:**
- If `event_distance < 0` → set to 0 (odometer reset)
- If `event_distance > 50,000m` → set to 0 (data error)
- If `event_fuel < 0` → set to 0 (sensor error)

### Step 3: Clutch Riding Detection
Flag events where ALL conditions are met:
```python
is_clutch_riding = (
    clutch_switch_status == 'Pressed' AND
    vehiclespeed > 20 AND
    accelerator_pedal_pos > 2
)
```

### Step 4: Calculate Statistics
Split data by `is_clutch_riding` flag:
- **Clutch Riding Events**: where flag = TRUE
- **Normal Riding Events**: where flag = FALSE
- **Overall**: All events

For each category, calculate:
- Total distance (sum of event_distance, convert to km)
- Total fuel (sum of event_fuel, in liters)
- Mileage = distance_km / fuel_liters

---

## 5. Output Table Structure

### Final Table: `clutch_riding_daily_summary`

| Column | Type | Description | Unit |
|--------|------|-------------|------|
| `analysis_date` | DATE | Date of analysis | YYYY-MM-DD |
| `uniqueid` | VARCHAR | Vehicle identifier | - |
| **Overall Metrics** | | | |
| `overall_distance_km` | FLOAT | Total distance traveled | km |
| `overall_fuel_liters` | FLOAT | Total fuel consumed | liters |
| `overall_mileage_kmpl` | FLOAT | Overall fuel efficiency | km/L |
| **Clutch Riding Metrics** | | | |
| `clutch_riding_distance_km` | FLOAT | Distance with clutch riding | km |
| `clutch_riding_fuel_liters` | FLOAT | Fuel during clutch riding | liters |
| `clutch_riding_mileage_kmpl` | FLOAT | Mileage during clutch riding | km/L |
| **Normal Riding Metrics** | | | |
| `normal_riding_distance_km` | FLOAT | Distance without clutch riding | km |
| `normal_riding_fuel_liters` | FLOAT | Fuel during normal riding | liters |
| `normal_riding_mileage_kmpl` | FLOAT | Mileage during normal riding | km/L |
| **Percentages** | | | |
| `clutch_riding_distance_pct` | FLOAT | % of distance in clutch riding | % |
| `clutch_riding_fuel_pct` | FLOAT | % of fuel used in clutch riding | % |
| **Savings** | | | |
| `fuel_savings_possible_liters` | FLOAT | Potential fuel savings | liters |
| `monetary_savings_inr` | FLOAT | Potential money savings | ₹ |

---

## 6. Savings Calculation Logic

```python
# Calculate how much fuel SHOULD have been used
fuel_should_have_used = clutch_riding_distance_km / normal_riding_mileage_kmpl

# Calculate wasted fuel
fuel_wasted = clutch_riding_fuel_liters - fuel_should_have_used

# Potential savings
fuel_savings_possible_liters = max(0, fuel_wasted)
monetary_savings_inr = fuel_savings_possible_liters × 100  # ₹100/liter
```

**Interpretation:** If the driver had maintained normal riding efficiency during clutch riding segments, they would have saved X liters per day.

---

## 7. Example Output

| analysis_date | uniqueid | overall_distance_km | overall_fuel_liters | overall_mileage_kmpl | clutch_riding_distance_km | clutch_riding_fuel_liters | clutch_riding_mileage_kmpl | normal_riding_distance_km | normal_riding_fuel_liters | normal_riding_mileage_kmpl | clutch_riding_distance_pct | clutch_riding_fuel_pct | fuel_savings_possible_liters | monetary_savings_inr |
|---------------|----------|---------------------|---------------------|----------------------|---------------------------|---------------------------|----------------------------|---------------------------|---------------------------|----------------------------|----------------------------|------------------------|------------------------------|----------------------|
| 2026-01-15 | VEH001 | 185.5 | 15.2 | 12.2 | 28.4 | 3.5 | 8.1 | 157.1 | 11.7 | 13.4 | 15.3 | 23.0 | 1.38 | 138.00 |
| 2026-01-15 | VEH002 | 165.0 | 18.5 | 8.9 | 62.3 | 9.8 | 6.4 | 102.7 | 8.7 | 11.8 | 37.8 | 53.0 | 4.52 | 452.00 |
| 2026-01-15 | VEH003 | 142.8 | 12.1 | 11.8 | 5.2 | 0.6 | 8.7 | 137.6 | 11.5 | 12.0 | 3.6 | 5.0 | 0.17 | 17.00 |

---

## 8. Data Quality Filters

### Exclusion Criteria (vehicles NOT included in final table):

| Criterion | Threshold | Reason |
|-----------|-----------|--------|
| Sample size | < 50 records | Insufficient data |
| Fuel data quality | > 60% NULL | Cannot calculate mileage |
| Clutch data quality | > 50% NULL | Cannot detect clutch riding |
| Distance | < 10 km | Not statistically significant |

---

## 9. Storage & Outputs

### Output Files Generated:

1. **`clutch_riding_statistics.xlsx`** - Complete statistics (all columns above)
2. **`mileage_summary_filtered_10km.xlsx`** - Filtered for vehicles with >10 km distance
3. **`dqm_report.xlsx`** - Data quality report
4. **`failed_vehicles.xlsx`** - Vehicles excluded with reasons

### File Format:
- Excel (.xlsx) format
- One row per vehicle per day
- Generated daily at 03:00 AM
- Retention: 90 days

---

## 10. Technical Configuration

| Parameter | Value |
|-----------|-------|
| Parallel workers | 10 |
| Database pool size | 20 connections |
| Retry attempts | 3 |
| Processing mode | Per-vehicle, per-day |
| Date processing order | Newest to oldest |

---

## 11. Key Formulas

```python
# Event calculation
event_distance[i] = obddistance[i+1] - obddistance[i]
event_fuel[i] = fuel_consumption[i+1] - fuel_consumption[i]

# Mileage calculation
mileage_kmpl = (sum(event_distance) / 1000) / sum(event_fuel)

# Savings calculation
savings_liters = clutch_fuel - (clutch_distance / normal_mileage)
savings_inr = savings_liters × 100
```

---

## 12. Success Criteria

**Per Vehicle:**
- ✅ Daily statistics generated
- ✅ Clutch riding distance, fuel, and mileage calculated
- ✅ Savings potential quantified
- ✅ Stored in final table

**Fleet-Wide:**
- ✅ > 85% of vehicles successfully processed
- ✅ All output files generated
- ✅ Processing completes in < 10 minutes

---

**End of Document**
