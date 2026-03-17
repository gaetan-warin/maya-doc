# Water Epic 341 — Go-Live Procedure

**Date:** 2026-03-17 | **Relates to:** IMPLEMENTATION_PLAN.md, LYNX_COMPLETE_REFERENCE.md

---

## Pre-requisites

| # | Check | Status |
|---|-------|--------|
| 1 | Backend code deployed (Lynx ingest, promotion, health + connected meter admin) | [ ] |
| 2 | Back office deployed (connected device form + Lynx club config form) | [ ] |
| 3 | Water Page frontend deployed (Lynx display changes) | [ ] |
| 4 | ETL code deployed (`pipelines/lynx/` + `masgrau.py` + `shayp.py` + `lib/water.py`) | [ ] |
| 5 | ETL `.env` has all `LYNX_*` vars configured | [ ] |
| 6 | Core `.env` has `SLACK_LYNX_WEBHOOK_URL` configured | [ ] |
| 7 | Lynx SQL Server reachable from ETL host (club's local network) | [ ] |
| 8 | ODBC Driver 17 for SQL Server installed on ETL host | [ ] |

---

## Part A — Connected Meters (Masgrau + Shayp)

### A.1. Run migration — Seed water sources, devices & site associations

**Migration:** `2026_03_13_000001_seed_masgrau_water_sources_and_devices.php`

```bash
php artisan migrate --path=database/migrations/2026_03_13_000001_seed_masgrau_water_sources_and_devices.php
```

Creates 3 water_sources + 3 connected_water_meter_devices + 3 site associations for Infinitum Living.

**Target tenant:** `a8b94cd9-2421-46ea-bfd2-71da150ad027`
**Target site:** `37dcd69b-b9c9-478c-92b2-9680a81ebcf0`

- [x] Done (dev — 2026-03-13)

### A.2. Update Masgrau ETL schedule

Change from every 5 minutes to **every 60 minutes** (at :05 past each hour).

- [ ] Done

---

## Part B — Lynx Integration

### B.1. Run Lynx migrations

```bash
php artisan migrate --path=database/migrations/2026_03_17_100001_create_lynx_club_configs_table.php
php artisan migrate --path=database/migrations/2026_03_17_100002_create_lynx_water_records_table.php
php artisan migrate --path=database/migrations/2026_03_17_100003_create_lynx_sync_logs_table.php
php artisan migrate --path=database/migrations/2026_03_17_100004_add_is_lynx_record_to_water_readings_table.php
php artisan migrate --path=database/migrations/2026_03_17_100005_add_site_id_to_lynx_club_configs_table.php
```

**Creates:** `lynx_club_configs`, `lynx_water_records`, `lynx_sync_logs` tables + `is_lynx_record` column on `water_readings` + `site_id` column on `lynx_club_configs`.

**Rollback:** `php artisan migrate:rollback --step=5`

- [x] Done (cloud dev DB — 2026-03-17)

### B.2. Configure Slack webhook

Add to core-2.0 `.env`:

```env
SLACK_LYNX_WEBHOOK_URL=https://hooks.slack.com/services/XXXXX/XXXXX/XXXXX
```

Used by `lynx:check-sync-health` command to alert stale clubs. Create the webhook in Slack workspace under a `#lynx-alerts` channel.

- [ ] Done

### B.3. Create Adare Manor club config via back office

1. Open back office → `/water/lynx`
2. Click **Add Club**
3. Fill in:
   - **Tenant ID:** Adare Manor tenant UUID
   - **Site ID:** Adare Manor site UUID
   - **Club Slug:** `adare-manor`
   - **Unit:** `m3`
   - **Timezone:** `Europe/Dublin`
   - **Day Start:** `15:55`
4. Copy the generated API key — it won't be shown again

- [ ] Done

### B.4. Configure Lynx ETL env vars

Add to ETL `.env`:

```env
# Lynx — Adare Manor
LYNX_DB_HOST=192.168.x.x
LYNX_DB_PORT=1433
LYNX_DB_NAME=lynx_main
LYNX_DB_USER=maya_reader
LYNX_DB_PASSWORD=CHANGE_ME
LYNX_DB_DRIVER=ODBC Driver 17 for SQL Server
LYNX_API_URL=https://api2.mayaglobal.io/api/v2/lynx/sync
LYNX_API_KEY=lynx_PASTE_KEY_FROM_STEP_B3
LYNX_CLUB_IDENTIFIER=adare-manor
LYNX_UNIT=m3
LYNX_IRRIGATION_DAY_START=15:55
```

- [ ] Done

### B.5. Test Lynx pipeline — dry run

```bash
cd /path/to/ETL
python pipelines/lynx/main.py --dry-run --verbose
```

Expected: JSON output with reconciled zone records. No API call made.

- [ ] Done

### B.6. Test Lynx pipeline — live push

```bash
python pipelines/lynx/main.py --verbose
```

Expected: `Sync complete: status=success, accepted=N, rejected=0`

Verify in DB:

```sql
SELECT * FROM lynx_water_records ORDER BY created_at DESC LIMIT 20;
SELECT * FROM lynx_sync_logs ORDER BY created_at DESC LIMIT 5;
```

- [ ] Done

### B.7. Test promotion

Trigger manually:

```bash
php artisan water:promote-lynx-daily-readings
```

Verify water_readings created:

```sql
SELECT wr.*, ws.name AS source_name
FROM water_readings wr
JOIN water_sources ws ON wr.water_source_id = ws.id
WHERE wr.is_lynx_record = 1
ORDER BY wr.reading_date DESC
LIMIT 20;
```

- [ ] Done

### B.8. Test health check

```bash
php artisan lynx:check-sync-health
```

Expected: "All clubs healthy" (if B.6 ran recently) or Slack alert (if >26h since last sync).

- [ ] Done

### B.9. Schedule Lynx pipeline

Add to ETL scheduler (APScheduler in `app/scheduler.py`) or system cron:

```
# Daily at 06:00 local time (before 06:30 promotion service)
0 6 * * * cd /path/to/ETL && python pipelines/lynx/main.py >> logs/lynx_cron.log 2>&1
```

- [ ] Done

### B.10. Optional: historical backfill

```bash
python pipelines/lynx/main.py --backfill --months 12 --verbose
```

Uses `water_use` table (permanent, but midnight boundaries — less precise than actuals). Run once for historical data.

- [ ] Done (if applicable)

---

## Part C — Scheduled Commands (DevOps)

Three artisan commands declared in `routes/console.php`. **DevOps must confirm the Laravel scheduler is running.**

| Command | Schedule | Purpose |
|---------|----------|---------|
| `water:aggregate-connected-daily-readings` | Hourly | `connected_water_meter_hourly_records` → `water_readings` (Masgrau + Shayp) |
| `water:promote-lynx-daily-readings` | Daily 06:30 | `lynx_water_records` → `water_readings` (Lynx) |
| `lynx:check-sync-health` | Daily 08:00 | Flag clubs with no sync in >26h → Slack alert |

**For DevOps:** ensure `php artisan schedule:run` runs every minute on the production server (standard Laravel cron), or `php artisan schedule:work` runs as a persistent process.

```cron
* * * * * cd /path/to/core-2.0 && php artisan schedule:run >> /dev/null 2>&1
```

- [ ] DevOps confirmed scheduler is running in production

---

## Part D — Verification

### D.1. Water Dashboard — visual check

Open the Water Dashboard for the target tenant and confirm:

1. Masgrau data appears in daily + hourly views
2. Lynx zone data appears in the outflows list
3. Lynx records show purple link icon in Water Records table
4. Calendar dots are green + locked (protected) for Lynx irrigation days
5. Hourly toggle is NOT shown for Lynx outflows

- [ ] Done

---

## Production Changes Log

| Date | Change | Who | Ticket | Rollback |
|------|--------|-----|--------|----------|
| | Migration: seed Masgrau water sources + devices + site associations | | | `php artisan migrate:rollback --path=...seed_masgrau...` |
| | Cron: Masgrau schedule 5min → 60min | | | Revert cron entry |
| | Migration: Lynx tables (5 migrations) | | | `php artisan migrate:rollback --step=5` |
| | Deploy: core-2.0 (Lynx ingest + promotion + health + back office API + Water Page API) | | | Redeploy previous version |
| | Deploy: back-office (connected device + Lynx club config forms) | | | Redeploy previous version |
| | Deploy: web (Water Page Lynx display) | | | Redeploy previous version |
| | Deploy: ETL (`pipelines/lynx/` + `lib/water.py` + `masgrau.py` + `shayp.py`) | | | Redeploy previous version |
| | Config: `.env` — `SLACK_LYNX_WEBHOOK_URL` on core-2.0 | | | Remove var |
| | Config: `.env` — `LYNX_*` vars on ETL host | | | Remove vars |
| | DevOps: Laravel scheduler running in production | | | N/A |
| | Lynx pipeline: daily cron at 06:00 on ETL host | | | Remove cron entry |
| | Club config: Adare Manor created via back office | | | Delete via back office |

---

## Rollback Plan

### Connected Meters

1. Stop Masgrau ETL cron
2. Rollback seed migration
3. Redeploy previous ETL + backend

### Lynx

1. Stop Lynx pipeline cron
2. Rollback 5 Lynx migrations (cascading deletes clean up all Lynx data)
3. Redeploy previous backend + ETL
4. Water Page reverts automatically (no Lynx data → no Lynx display)

Shayp pipeline is unaffected — fully independent.
