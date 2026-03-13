# Connected Water Meter Unification Plan

**Date:** 2026-03-13 | **Scope:** Masgrau + Shayp unified on Water API | **Relates to:** Phase 2 of IMPLEMENTATION_PLAN.md

---

## Problem Statement

Masgrau and Shayp are both water meters measuring consumption, but they use completely different data paths:

| | Shayp (current) | Masgrau (current) |
|---|---|---|
| Data type | Hourly consumption (litres) | Cumulative meter reading (raw 32-bit) |
| ETL endpoint | `POST /api/v2/water/hourly-records` | `POST /api/odata/v1/receiveMetricData` |
| DB table | `connected_water_meter_hourly_records` | `metric` (legacy) |
| Feeds water dashboard? | Yes | **No** |
| Feeds daily aggregation? | Yes | **No** |
| Other metrics sent | None (water only) | 30+ unused metrics (pump amps, frequency, pressure, pH, conductivity, alarms) |

**Result:** Masgrau water consumption data is a dead end — stored in the `metric` table but never processed, never shown on the water dashboard, never aggregated.

Additionally, none of the non-water Masgrau metrics (pump data, pressure, pH, alarms) are displayed anywhere in the Maya app. They are stored but unused.

---

## Design Principle: Stateless ETL

The ETL should be a **dumb pipe** — read from source, post to API, done. No state, no delta calculation, no dependency on previous runs.

- **ETL responsibility:** Read Modbus registers, convert units, POST cumulative value to Water API
- **Backend responsibility:** Own all business logic — detect measurement type, compute deltas, aggregate daily

This keeps the integration stateless and makes the backend the single source of truth for data processing.

---

## Target Architecture

```
          ┌─────────────────┐          ┌─────────────────┐
          │  Masgrau PLC    │          │  Shayp API      │
          │  (Modbus/TCP)   │          │                 │
          └────────┬────────┘          └────────┬────────┘
                   │ Every hour                 │ Every hour
                   ▼                            ▼
          ┌─────────────────┐          ┌─────────────────┐
          │  maya-etl       │          │  maya-etl       │
          │  masgrau.py     │          │  shayp.py       │
          │                 │          │                 │
          │  POST cumulative│          │  POST delta     │
          │  m3 (stateless) │          │  m3 (stateless) │
          └────────┬────────┘          └────────┬────────┘
                   │                            │
                   ▼                            ▼
          ┌──────────────────────────────────────────────┐
          │  Water API: POST /water/hourly-records       │
          │  (accepts both cumulative and delta values)  │
          └──────────────────┬───────────────────────────┘
                             │ UPSERT
                             ▼
          ┌──────────────────────────────────────────────┐
          │  connected_water_meter_hourly_records         │
          │  (stores raw value — cumulative OR delta)     │
          └──────────────────┬───────────────────────────┘
                             │ Daily aggregation cron (hourly)
                             │
                             │  if measurement_type = daily_consumption:
                             │    SUM(consumption_value) for the day
                             │
                             │  if measurement_type = meter_reading:
                             │    last_record - first_record for the day
                             │
                             ▼
          ┌──────────────────────────────────────────────┐
          │  water_readings (one row per source per day)  │
          └──────────────────┬───────────────────────────┘
                             │ allocation via default associations
                             ▼
          ┌──────────────────────────────────────────────┐
          │  water_site_consumption (allocated to sites)  │
          └──────────────────┬───────────────────────────┘
                             │
                             ▼
          ┌──────────────────────────────────────────────┐
          │  Water Dashboard (hourly + daily views)       │
          └──────────────────────────────────────────────┘
```

---

## Step-by-Step Logic

### Step 1 — ETL reads Masgrau PLC (every 60 minutes)

`masgrau.py` connects via Modbus/TCP and reads 2 registers per pumping group:

```
WaterConsumption_Low  (register 3x0Y13)
WaterConsumption_High (register 3x0Y14)
```

Combines into 32-bit value: `High * 65536 + Low`

Converts to m3: `raw_value / 1_000_000`

**Stop sending** all other metrics (pressure, amps, frequency, pH, etc.) — none are used.

### Step 2 — ETL posts cumulative m3 to Water API (stateless)

For each pumping group, POST to `/api/v2/water/hourly-records`:

```json
{
  "device_id": 123,
  "records": [
    {
      "consumption_value": 5.935770,
      "date_time": "2026-03-13 14:00:00"
    }
  ]
}
```

The `consumption_value` here is the **cumulative** m3 reading — not a delta. The ETL doesn't know or care about previous readings. It just posts what it reads.

### Step 3 — Water API upserts into hourly records

The existing endpoint validates and upserts into `connected_water_meter_hourly_records` using the unique constraint `(device_id, date_time)`. No change needed here.

### Step 4 — Daily aggregation cron processes hourly records

The `AggregateConnectedWaterMeterDailyReadings` command runs hourly. Currently it only handles `daily_consumption` sources (SUM of hourly values).

**New behavior** — branch on `measurement_type` of the water source:

| measurement_type | Aggregation logic | Example |
|---|---|---|
| `daily_consumption` (Shayp) | `SUM(consumption_value)` for all hours in the day | 0.38 + 0.42 + 0.15 + ... = 4.2 m3 |
| `meter_reading` (Masgrau) | `last_record.consumption_value - first_record.consumption_value` for the day | 5.94 - 5.82 = 0.12 m3 |

This is the **only backend code change** needed.

### Step 5 — Water reading is created

The aggregation service calls `WaterReadingService::createWaterReading()` with:
- `reading_value` = daily consumption (computed delta for meter_reading, sum for daily_consumption)
- `consumption_value` = same value
- `measurement_type` = from the water source
- `is_connected_device_record` = true

### Step 6 — Site consumption is allocated

Using `water_source_default_associations`, the consumption is distributed to linked sites → `water_site_consumption` table. Existing logic, no changes.

### Step 7 — Dashboard displays data

The Water Dashboard already renders data from `water_site_consumption` + `connected_water_meter_hourly_records` for hourly drill-down. No frontend changes.

---

## Changes Required

### 1. ETL — `maya-etl`

#### 1.1 Simplify `pipelines/masgrau.py`

| What | Detail |
|------|--------|
| **Remove** | All non-water metric reads (pressure, amps, frequency, pH, conductivity, alarms) |
| **Remove** | POST to `receiveMetricData` — no more metric table writes |
| **Keep** | Modbus read of `WaterConsumption_Low` + `WaterConsumption_High` per PG |
| **Add** | Unit conversion: `High * 65536 + Low` then `/ 1_000_000` = cumulative m3 |
| **Add** | POST cumulative m3 to `/api/v2/water/hourly-records` |

The ETL becomes ~30 lines of logic: read 6 registers (2 per PG), convert, POST. No state, no delta, no DB queries.

#### 1.2 Add Water API posting function

New function in `lib/water.py`: `post_water_hourly_record(device_id, consumption_value, date_time)`

Reuse existing env vars from Shayp:
- `MAYA_WATER_API_URL`
- `MAYA_WATER_JWT`

#### 1.3 Update schedule

From every 5 minutes → **every 60 minutes** (at :05 past each hour). Matches Shayp cadence.

#### 1.4 Configuration (`.env`)

```
# Masgrau Modbus connection (already exist)
MASGRAU_HOST=95.129.253.218
MASGRAU_PORT=502

# Connected Water Meter device IDs (new, assigned after migration)
MASGRAU_DEVICE_ID_PG1=<to be assigned>
MASGRAU_DEVICE_ID_PG2=<to be assigned>
MASGRAU_DEVICE_ID_PG3=<to be assigned>

# Already exist (shared with Shayp):
# MAYA_WATER_API_URL=https://api2.mayaglobal.io/api/v2/water/hourly-records
# MAYA_WATER_JWT=<Bearer token>
```

---

### 2. Backend — `mayaApp/core-2.0`

#### 2.1 Modify aggregation service to support `meter_reading`

**File:** `app/Services/Water/ConnectedWaterMeterDailyAggregationService.php`

**Current:** Skips sources where `measurement_type !== daily_consumption`

**New:** Handle both types:

```php
if ($waterSource->measurement_type === WaterMeasurementType::DAILY_CONSUMPTION) {
    // Existing: SUM(consumption_value) for the day
    $dailyConsumption = $this->sumHourlyRecords($deviceId, $date);
} elseif ($waterSource->measurement_type === WaterMeasurementType::METER_READING) {
    // New: last_record - first_record for the day
    $dailyConsumption = $this->deltaFromCumulativeRecords($deviceId, $date);
}
```

The `deltaFromCumulativeRecords` method:
1. Query hourly records for device + date, ordered by `date_time`
2. `delta = last.consumption_value - first.consumption_value`
3. If delta < 0 (counter reset): log warning, use 0 or skip
4. If only 1 record: delta = 0 (can't compute)

#### 2.2 Create migration: water sources + devices + site associations

**Water sources** (3 outflows):

| Field | PG1 (Nord) | PG2 (Centre) | PG3 (Sud) |
|-------|-----------|-------------|-----------|
| `tenant_id` | `a8b94cd9-2421-46ea-bfd2-71da150ad027` | same | same |
| `name` | Masgrau PG1 - Nord | Masgrau PG2 - Centre | Masgrau PG3 - Sud |
| `source_type` | `outflow` | `outflow` | `outflow` |
| `measurement_type` | `meter_reading` | `meter_reading` | `meter_reading` |
| `is_legacy` | `false` | `false` | `false` |

**Connected water meter devices** (3 devices):

| Field | PG1 (Nord) | PG2 (Centre) | PG3 (Sud) |
|-------|-----------|-------------|-----------|
| `device_reference_id` | `masgrau-infinitum-pg1` | `masgrau-infinitum-pg2` | `masgrau-infinitum-pg3` |
| `tenant_id` | `a8b94cd9-2421-46ea-bfd2-71da150ad027` | same | same |
| `water_source_id` | FK → PG1 water source | FK → PG2 water source | FK → PG3 water source |
| `status` | 1 (active) | 1 (active) | 1 (active) |

**Site associations** (3 rows):

| Field | Value |
|-------|-------|
| `water_source_id` | FK → each of the 3 water sources |
| `site_id` | Infinitum Living site UUID (to be queried) |
| `is_active` | `true` |

#### 2.3 No API endpoint changes

The existing Water API endpoint accepts `consumption_value` as a number and upserts. It doesn't validate whether the value is cumulative or delta — it just stores it. No change needed.

---

### 3. No Frontend Changes

The dashboard reads from `water_site_consumption` and `connected_water_meter_hourly_records`. Both tables are populated by the existing pipeline. No UI changes.

---

## Edge Cases

| Case | Where handled | How |
|------|---------------|-----|
| **First day** (only 1 hourly record) | Backend aggregation | delta = 0, skip or create reading with 0 |
| **Counter reset** (PLC reboot) | Backend aggregation | last - first < 0 → log warning, use 0 |
| **PLC unreachable** | ETL | Existing Modbus error handling, no record posted |
| **Duplicate posts** | Water API | UPSERT on `(device_id, date_time)` — idempotent |
| **Missing hours** | Backend aggregation | Still works: delta = last available - first available |
| **Unit mismatch** | ETL | Masgrau: raw ÷ 1M = m3. Shayp: litres ÷ 1000 = m3. Both end up as m3. |

---

## Unit Conversion

### Masgrau (cumulative)
```
Raw 16-bit registers: WaterConsumption_Low (3x0Y13) + WaterConsumption_High (3x0Y14)
Combined 32-bit:      High * 65536 + Low
To m3:                combined_value / 1_000_000
Example:              90 * 65536 + 42630 = 5,935,770 → 5.935770 m3 (cumulative)
```

### Shayp (delta)
```
Raw:    quantity = 380 (litres, hourly consumption)
To m3:  380 / 1000 = 0.38 m3
```

### Water API payload (same for both)
```json
{
  "device_id": 123,
  "records": [
    {
      "consumption_value": 5.935770,
      "date_time": "2026-03-13 14:00:00"
    }
  ]
}
```

Note: For Masgrau this is cumulative m3. For Shayp this is hourly delta m3. The backend knows which is which via `water_sources.measurement_type`.

---

## Implementation Order

| Step | What | Where | Status |
|------|------|-------|--------|
| **1** | Get Infinitum Living `site_id` from DB | DB query | [x] `37dcd69b-b9c9-478c-92b2-9680a81ebcf0` |
| **2** | Add `meter_reading` support to aggregation service | `core-2.0` | [x] `ConnectedWaterMeterDailyAggregationService` updated + 2 bugfixes (`$measurementType->value`, `is_connected_device_record`) |
| **3** | Create migration: 3 water sources + 3 devices + 3 site associations | `core-2.0` | [x] Migration created & ran on dev |
| **4** | Add `device_reference_id` support to Water API | `core-2.0` | [x] Request, service, repository updated |
| **5** | Simplify `masgrau.py`: stateless, POST cumulative via `device_reference_id` | `maya-etl` | [x] Rewritten |
| **6** | Create shared `lib/water.py` (POST to Water API) | `maya-etl` | [x] Extracted from shayp.py |
| **7** | Update `shayp.py` to use `device_reference_id` + shared `lib/water.py` | `maya-etl` | [x] Updated |
| **8** | Update cron schedule from 5min to 60min | ETL server cron | [ ] Deployment-time task |
| **9** | Test: POST with `device_reference_id` to Water API | Manual test | [x] 201 for all 3 PGs (local Docker) |
| **10** | Test: hourly data appears in `connected_water_meter_hourly_records` | Manual test | [x] 3 records confirmed in DB |
| **11** | Test: daily aggregation computes correct delta | Manual test | [x] 3 water_readings + 3 site_consumption created (local Docker, --force). Fixed bug: `$measurementType->value` → `$measurementType` |
| **12** | Test: data appears on Water Dashboard | Manual test | [x] API returns 69 connected meter readings. Fixed bug: `is_connected_device_record` not set by aggregation service |

---

## Open Questions

| # | Question | Owner | Status |
|---|----------|-------|--------|
| 1 | What is the Infinitum Living `tenant_id` UUID? | Backend/DB | **Resolved: `a8b94cd9-2421-46ea-bfd2-71da150ad027`** |
| 2 | Should each PG be a separate water source, or all 3 feed into one source? | Product | **Decided: 3 separate** |
| 3 | Are the 3 PGs inflows or outflows? | Product | **Decided: Outflows** — Masgrau pumps measure irrigation water volume, same as Shayp ("Arrosage"). Direction is a physical/business decision, not in the Modbus payload. |
| 4 | Confirm unit: raw 32-bit / 1,000,000 = m3? | ETL/Masgrau vendor | **Confirmed** — per Notion doc: "The values should be in Millions, which gives the water meter reading" |
| 5 | Should we backfill historical Masgrau data from the metric table? | Product | **Decided: No** |
| 6 | What is the Infinitum Living `site_id` UUID? | Backend/DB | **Resolved: `37dcd69b-b9c9-478c-92b2-9680a81ebcf0`** (site 20 on demo tenant) |

---

## Relationship to Existing Plan

This work falls under **Phase 2 — Verify Connected Meters** in `IMPLEMENTATION_PLAN.md`:

- **2.1** "Verify Shayp/Masgrau data loads correctly on Water Page" — this plan makes Masgrau actually work
- **5.2** "Prioritize Shayp / Masgrau / others by customer demand" — this plan executes the Masgrau priority

Estimated effort: **3-4 dev days** (Steps 1-11)
