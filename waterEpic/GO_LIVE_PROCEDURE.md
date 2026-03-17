# Connected Water Meter — Go-Live Procedure

**Date:** 2026-03-13 | **Relates to:** CONNECTED_METER_UNIFICATION_PLAN.md

---

## Pre-requisites

| # | Check | Status |
|---|-------|--------|
| 1 | Backend code deployed (aggregation service + `device_reference_id` support) | [ ] |
| 2 | ETL code deployed (`masgrau.py` stateless, `shayp.py` updated, `lib/water.py` shared) | [ ] |
| 3 | ETL `.env` has `MAYA_WATER_API_URL` and `MAYA_WATER_JWT` | [ ] |
| 4 | Masgrau PLC reachable from ETL host (`95.129.253.218:502`) | [ ] |

---

## Step-by-step Go-Live

### 1. Run migration — Seed water sources, devices & site associations

**Migration:** `2026_03_13_000001_seed_masgrau_water_sources_and_devices.php`

```bash
cd /path/to/core-2.0
php artisan migrate --path=database/migrations/2026_03_13_000001_seed_masgrau_water_sources_and_devices.php
```

**What it creates:**

| Table | Rows | Details |
|-------|------|---------|
| `water_sources` | 3 | PG1 Nord, PG2 Centre, PG3 Sud — `measurement_type=meter_reading`, `source_type=outflow` |
| `connected_water_meter_devices` | 3 | `masgrau-infinitum-pg1`, `pg2`, `pg3` linked to water sources |
| `water_source_default_associations` | 3 | Links each source to site |

**Target tenant:** `a8b94cd9-2421-46ea-bfd2-71da150ad027`
**Target site:** `37dcd69b-b9c9-478c-92b2-9680a81ebcf0`

**Note:** Uses `UUID_TO_BIN($uuid, 1)` — the DB stores UUIDs with byte swap.

**Note:** No `.env` device ID configuration needed — ETL uses `device_reference_id` (e.g. `masgrau-infinitum-pg1`) which the API resolves automatically.

**Rollback:**
```bash
php artisan migrate:rollback --path=database/migrations/2026_03_13_000001_seed_masgrau_water_sources_and_devices.php
```

- [x] Done (dev — 2026-03-13)
- [x] Verified in DB

---

### 2. Update cron schedule

Change Masgrau ETL schedule from every 5 minutes to **every 60 minutes** (at :05 past each hour) to match Shayp cadence.

- [ ] Done

---

### 2b. Scheduled commands — DevOps action required

Two artisan commands are declared in `routes/console.php` and must be running in production. **DevOps is responsible for ensuring the Laravel scheduler is running on the server.** Dev team owns the schedule declaration only.

**Commands declared in `routes/console.php`:**

| Command | Schedule | Purpose |
|---------|----------|---------|
| `water:aggregate-connected-daily-readings` | Hourly | Converts `connected_water_meter_hourly_records` → daily `water_readings` (Masgrau + Shayp) |
| `water:promote-lynx-daily-readings` *(to be added)* | Daily at 06:30 | Promotes `lynx_water_records` → `water_readings` (Lynx) |

**For DevOps:** ensure `php artisan schedule:run` is triggered every minute on the production server (standard Laravel cron entry), or that `php artisan schedule:work` is running as a persistent process.

- [ ] DevOps confirmed scheduler is running in production

---

### 3. Test: POST with device_reference_id to Water API

Send a test POST to verify the backend resolves `device_reference_id` correctly:

```bash
curl -X POST /api/v2/water/hourly-records \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "device_reference_id": "masgrau-infinitum-pg1",
    "records": [{"consumption_value": 1.0, "date_time": "2026-03-13 10:00:00"}]
  }'
```

Expected: 201 Created.

- [x] Done (local Docker — 2026-03-13) — All 3 PGs return 201, resolved to device_id 5/6/7

---

### 4. Test: ETL posts data

Run masgrau.py manually once and check:

```sql
SELECT hr.*, d.device_reference_id
FROM connected_water_meter_hourly_records hr
JOIN connected_water_meter_devices d ON hr.connected_water_meter_device_id = d.id
WHERE d.device_reference_id LIKE 'masgrau-infinitum-%'
ORDER BY hr.date_time DESC
LIMIT 10;
```

Expected: 3 rows (one per PG) with cumulative m3 values.

- [x] Done (local Docker — 2026-03-13) — 3 records: pg1=123.456, pg2=456.789, pg3=789.012 m3

---

### 5. Test: daily aggregation

Wait for the `AggregateConnectedWaterMeterDailyReadings` cron to run (or trigger manually), then check:

```sql
SELECT wr.*
FROM water_readings wr
JOIN water_sources ws ON wr.water_source_id = ws.id
WHERE ws.name LIKE 'Masgrau%'
ORDER BY wr.reading_date DESC;
```

Expected: daily readings with `measurement_type=meter_reading`, `consumption_value` = delta from cumulative.

- [x] Done (local Docker — 2026-03-13) — 3 water_readings created (PG1=123.456, PG2=456.789, PG3=789.012). Bug fixed: `$measurementType->value` → `$measurementType` in aggregation service line 219.

---

### 6. Test: site consumption allocation

```sql
SELECT wsc.*
FROM water_site_consumption wsc
JOIN water_readings wr ON wsc.water_reading_id = wr.id
JOIN water_sources ws ON wr.water_source_id = ws.id
WHERE ws.name LIKE 'Masgrau%'
ORDER BY wsc.created_at DESC;
```

- [x] Done (local Docker — 2026-03-13) — 3 water_site_consumption records allocated

---

### 7. Test: Water Dashboard

Open the Water Dashboard for Infinitum Living and confirm Masgrau data appears in both daily and hourly views.

- [x] API verified (local Docker — 2026-03-13) — 69 connected meter readings returned (3 Masgrau + 66 Shayp). Bug fixed: aggregation was not setting `is_connected_device_record=true`, so "Connected Meter Logs" tab was empty. Fixed in `ConnectedWaterMeterDailyAggregationService` + backfilled 69 existing records.
- [x] Hourly view verified (local Docker — 2026-03-13) — All 3 Masgrau sources return hourly graph data with consumption deltas and cumulative tracking. Source icons (speedometer for meter_reading, wifi badge for connected) confirmed in code.
- [ ] Visual check in browser (pending)

---

## Production Changes Log

Track every production change here in chronological order.

| Date | Change | Who | Ticket | Rollback |
|------|--------|-----|--------|----------|
| | Migration: seed Masgrau water sources + devices + site associations | | | `php artisan migrate:rollback --path=...seed_masgrau...` |
| | Cron: Masgrau schedule 5min → 60min | | | Revert cron entry |
| | DevOps: Laravel scheduler running in production (`schedule:run` cron or `schedule:work`) | | | N/A |
| | Deploy: aggregation service `meter_reading` support | | | Redeploy previous version |
| | Deploy: Water API `device_reference_id` resolution | | | Redeploy previous version |
| | Deploy: masgrau.py stateless rewrite | | | Redeploy previous version |
| | Deploy: shayp.py `device_reference_id` update | | | Redeploy previous version |
| | Deploy: `lib/water.py` shared module | | | Redeploy previous version |

---

## Rollback Plan

If something goes wrong after go-live:

1. **Stop Masgrau ETL** — disable cron or stop the pipeline
2. **Rollback migration** — removes water sources, devices, and associations
3. **Redeploy previous ETL** — restores old masgrau.py posting to metric table
4. **Redeploy previous backend** — restores aggregation service without `meter_reading` support

Shayp pipeline is unaffected — `lib/water.py` extraction and `device_reference_id` are backward-compatible.
