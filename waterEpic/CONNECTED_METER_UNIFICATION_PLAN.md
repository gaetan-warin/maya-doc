# Connected Water Meter Unification Plan

**Date:** 2026-03-12 | **Scope:** Masgrau + Shayp unified on Water API | **Relates to:** Phase 2 of IMPLEMENTATION_PLAN.md

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

## Target Architecture

```
                    ┌─────────────────┐
                    │  Masgrau PLC    │
                    │  (Modbus/TCP)   │
                    └────────┬────────┘
                             │ Every hour
                             ▼
                    ┌─────────────────┐
                    │  maya-etl       │
                    │  masgrau.py     │
                    └───┬─────────┬───┘
                        │         │
          ┌─────────────▼──┐   ┌──▼──────────────────┐
          │ metric table   │   │ Water API            │
          │ (cumulative m3)│   │ /water/hourly-records│
          │ = state store  │   │ (hourly delta m3)    │
          └────────────────┘   └──────────┬───────────┘
                                          │
          ┌─────────────────┐             │
          │  Shayp API      │             │
          └────────┬────────┘             │
                   │ Every hour           │
                   ▼                      ▼
          ┌─────────────────┐   ┌─────────────────────┐
          │  maya-etl       │   │ connected_water_     │
          │  shayp.py       ├──►│ meter_hourly_records │
          └─────────────────┘   └──────────┬──────────┘
                                           │ Daily aggregation cron
                                           ▼
                                ┌─────────────────────┐
                                │   water_readings     │
                                └──────────┬──────────┘
                                           │
                                           ▼
                                ┌─────────────────────┐
                                │ water_site_          │
                                │ consumption          │
                                └──────────┬──────────┘
                                           │
                                           ▼
                                ┌─────────────────────┐
                                │  Water Dashboard     │
                                │  (hourly + daily)    │
                                └─────────────────────┘
```

---

## Changes Required

### 1. ETL — `maya-etl` (this repo)

#### 1.1 Modify `pipelines/masgrau.py`

**Current behavior:** Reads all 18 variables + 18 alarms per pumping group, sends everything to `receiveMetricData`.

**New behavior:**

| Step | Action |
|------|--------|
| **Read** | Read only `WaterConsumption_Low` + `WaterConsumption_High` per pumping group (PG1, PG2, PG3) |
| **Combine** | `raw_value = High * 65536 + Low` (32-bit unsigned) |
| **Convert** | `cumulative_m3 = raw_value / 1_000_000` (per Notion doc: "values should be in Millions") |
| **Store cumulative** | POST cumulative value to `receiveMetricData` → `metric` table (state store for next delta) |
| **Query previous** | Query `metric` table via `lib/db.py` for last WaterConsumption value for this pumping group |
| **Calculate delta** | `hourly_consumption_m3 = cumulative_m3 - previous_cumulative_m3` |
| **Validate delta** | If delta < 0 (counter reset) or previous is None (first run): skip posting to Water API |
| **POST delta** | POST `hourly_consumption_m3` to `/api/v2/water/hourly-records` with the device_id for this pumping group |

**Stop sending:** FilterInletPressure, FilterOutletPressure, AmpPump1-5, FreqPump1-5, RateOfFlow, PartialM3Flowmeter, WaterpH, WaterConductivity, LakeAspirationLevel, all 18 alarm coils. None are used in the app.

#### 1.2 Update schedule

**Current:** Every 5 minutes (via NiFi/cron)
**New:** Every 60 minutes (at :05 past each hour)

Matches Shayp cadence and aligns with hourly records granularity.

#### 1.3 Add `lib/db.py` query

New function: `get_last_water_consumption(counter_name)` — queries the `metric` table for the most recent value of a given WaterConsumption counter. Returns the cumulative m3 value, or `None` if no previous reading exists.

#### 1.4 Add Water API posting function

New function in `lib/common.py` or `lib/water.py`: `post_water_hourly_record(device_id, consumption_value, date_time)` — POSTs to `/api/v2/water/hourly-records` with Bearer token auth.

Reuse existing env vars already used by Shayp (`shayp.py`):
- `MAYA_WATER_API_URL` — already set in `.env` (`https://api2.mayaglobal.io/api/v2/water/hourly-records`)
- `MAYA_WATER_JWT` — already set in `.env` (long-lived JWT Bearer token)

#### 1.5 Configuration (`.env`)

Add new env vars (Water API URL and JWT already exist from Shayp):
```
# Masgrau Modbus connection
MASGRAU_HOST=95.129.253.218
MASGRAU_PORT=502

# Connected Water Meter device IDs (from connected_water_meter_devices table)
MASGRAU_DEVICE_ID_PG1=<to be assigned after migration>
MASGRAU_DEVICE_ID_PG2=<to be assigned after migration>
MASGRAU_DEVICE_ID_PG3=<to be assigned after migration>

# Already exist (shared with Shayp):
# MAYA_WATER_API_URL=https://api2.mayaglobal.io/api/v2/water/hourly-records
# MAYA_WATER_JWT=<Bearer token>
```

---

### 2. Maya API — `mayaApp/core-2.0` (backend)

#### 2.1 Create `water_sources` (3 outflows)

Each pumping group gets its own water source:

| Field | PG1 (Nord) | PG2 (Centre) | PG3 (Sud) |
|-------|-----------|-------------|-----------|
| `tenant_id` | `69601c19-1f91-476e-9e71-f6da047a64c0` | same | same |
| `name` | Masgrau PG1 - Nord | Masgrau PG2 - Centre | Masgrau PG3 - Sud |
| `source_type` | `outflow` | `outflow` | `outflow` |
| `measurement_type` | `daily_consumption` | `daily_consumption` | `daily_consumption` |
| `is_legacy` | `false` | `false` | `false` |

**Note:** `measurement_type` must be `daily_consumption` (not `meter_reading`) because the aggregation service (`AggregateConnectedWaterMeterDailyReadings`) only supports `daily_consumption` today. The ETL handles the cumulative→delta conversion.

#### 2.2 Create `connected_water_meter_devices` (3 devices)

| Field | PG1 (Nord) | PG2 (Centre) | PG3 (Sud) |
|-------|-----------|-------------|-----------|
| `device_reference_id` | `masgrau-infinitum-pg1` | `masgrau-infinitum-pg2` | `masgrau-infinitum-pg3` |
| `tenant_id` | `69601c19-1f91-476e-9e71-f6da047a64c0` | same | same |
| `water_source_id` | FK → PG1 water source | FK → PG2 water source | FK → PG3 water source |
| `status` | 1 (active) | 1 (active) | 1 (active) |

**Action:** Create a Laravel migration to insert water sources first, then devices linked to them.

#### 2.3 Create `water_source_default_associations` (link sources to sites)

Each water source must be linked to at least one site via the `water_source_default_associations` table. Without this, the daily aggregation creates `water_readings` but no `water_site_consumption` records — and data won't appear on the dashboard.

| Field | Value |
|-------|-------|
| `water_source_id` | FK → each of the 3 water sources |
| `site_id` | The Infinitum Living site UUID (query: `SELECT idsite FROM site WHERE idsite IN (SELECT idsite FROM device WHERE guiname = 'Infinitum Living')`) |
| `is_active` | `true` |

**Action:** Include in the same migration. Need to identify the correct `site_id` for Infinitum Living.

#### 2.4 No code changes needed

The existing Water API endpoint, hourly record upsert logic, and daily aggregation cron all work as-is. The only requirement is that the water sources, devices, and site associations exist in the database.

---

### 3. No Frontend Changes

Both Masgrau and Shayp data will flow through the same `connected_water_meter_hourly_records` → `water_readings` → `water_site_consumption` pipeline. The existing Water Dashboard UI already handles:
- Hourly view toggle (for sources with connected meters)
- Daily aggregated view
- Cumulative view
- Source icons for connected meter devices

---

## Edge Cases to Handle

| Case | How to handle |
|------|---------------|
| **First run** (no previous reading in metric table) | Skip delta calculation, only store cumulative. Next run will have a previous value. |
| **Counter reset** (PLC reboot, maintenance) | Delta would be negative. Detect and skip: `if delta < 0: log warning, skip Water API post, store new cumulative` |
| **PLC unreachable** (network issue) | Existing Modbus error handling in masgrau.py. No reading = no post. |
| **Duplicate hourly records** | Water API uses UPSERT on `(device_id, date_time)` — safe to re-post. |
| **Unit mismatch** | Masgrau: raw 32-bit ÷ 1,000,000 = m3. Shayp: litres ÷ 1,000 = m3. Both end up as m3. |

---

## Unit Conversion Details

### Masgrau
```
Raw 16-bit registers: WaterConsumption_Low (3x0Y13) + WaterConsumption_High (3x0Y14)
Combined 32-bit:      High * 65536 + Low
To m3:                combined_value / 1_000_000
Example:              90 * 65536 + 42630 = 5,935,770 → 5.935770 m3 (cumulative)
```

### Shayp
```
Raw:    quantity = 380 (litres, hourly consumption)
To m3:  380 / 1000 = 0.38 m3
```

### Water API expects
```json
{
  "device_id": 1,
  "records": [
    {
      "consumption_value": 0.38,   // m3, hourly consumption
      "date_time": "2026-03-12 14:00:00"  // UTC
    }
  ]
}
```

---

## Implementation Order

| Step | What | Where | Depends on |
|------|------|-------|------------|
| **Step 1** | Get Infinitum Living `site_id` from DB | DB query | — |
| **Step 2** | Create migration: 3 water sources + 3 devices + 3 site associations | `mayaApp` migration | Step 1 |
| **Step 3** | Run migration, note the assigned device IDs | `mayaApp` | Step 2 |
| **Step 4** | Add `get_last_water_consumption()` to `lib/db.py` | `maya-etl` | — |
| **Step 5** | Add `post_water_hourly_record()` to `lib/common.py` (reuse `MAYA_WATER_API_URL` + `MAYA_WATER_JWT`) | `maya-etl` | — |
| **Step 6** | Modify `masgrau.py`: only read WaterConsumption, calculate delta, dual-post | `maya-etl` | Steps 3-5 |
| **Step 7** | Update `.env` with device IDs (`MASGRAU_DEVICE_ID_PG1/PG2/PG3`) | `maya-etl` | Step 3 |
| **Step 8** | Update cron schedule from 5min to 60min | NiFi or cron config | Step 6 |
| **Step 9** | Verify data appears on Water Dashboard | Manual test | Steps 1-8 |
| **Step 10** | Verify daily aggregation picks up Masgrau data | Manual test | Step 9 |

---

## Open Questions

| # | Question | Owner | Status |
|---|----------|-------|--------|
| 1 | What is the Infinitum Living `tenant_id` UUID? | Backend/DB | **Resolved: `69601c19-1f91-476e-9e71-f6da047a64c0`** |
| 2 | Should each PG be a separate water source, or all 3 feed into one source? | Product | **Decided: 3 separate** |
| 3 | Are the 3 PGs inflows or outflows? | Product | **Decided: Outflows** — Shayp is configured as outflow in both prod ("Arrosage") and dev ("Shayp Outlet"). Masgrau pumps measure irrigation water volume, same concept → outflows. |
| 4 | Confirm unit: raw 32-bit ÷ 1,000,000 = m3? | ETL/Masgrau vendor | **Confirmed** — per Notion doc: "The values should be in Millions, which gives the water meter reading" |
| 5 | Should we backfill historical Masgrau data from the metric table? | Product | **Decided: No** |

---

## Relationship to Existing Plan

This work falls under **Phase 2 — Verify Connected Meters** in `IMPLEMENTATION_PLAN.md`:

- **2.1** "Verify Shayp/Masgrau data loads correctly on Water Page" — this plan makes Masgrau actually work
- **5.2** "Prioritize Shayp / Masgrau / others by customer demand" — this plan executes the Masgrau priority

Estimated effort: **3-4 dev days** (Steps 1-9)
