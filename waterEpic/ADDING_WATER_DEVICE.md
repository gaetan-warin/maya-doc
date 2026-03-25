# Adding a Water Device — End-to-End Guide

**Last updated:** 2026-03-18

This document explains how to add a new water device (connected meter or Lynx club) so that the ETL pipeline discovers it, syncs data, and the Water Page displays it.

---

## Architecture Overview

Three parallel data flows all converge into the `water_readings` table:

```
MASGRAU (Modbus/TCP)  ──→ hourly records ──→ daily aggregation ──→ water_readings
SHAYP   (HTTP API)    ──→ hourly records ──→ daily aggregation ──→ water_readings
LYNX    (SQL Server)  ──→ lynx_water_records ──→ promotion     ──→ water_readings
```

The ETL does NOT poll a list of "devices to sync". Instead:
- **Masgrau/Shayp**: The Python ETL agent pushes data referencing a `device_reference_id`. The backend resolves this to a `connected_water_meter_device` row.
- **Lynx**: The Python agent authenticates via API key, which maps to a `lynx_club_config` row.

**Creating the device in the back office is what makes the pipeline work.** Without it, pushed data has nowhere to land.

---

## Database Schema (relevant tables)

```
connected_water_meter_devices
├── id (PK)
├── device_reference_id    ← stable ID the ETL agent sends (e.g. "masgrau-infinitum-pg1")
├── tenant_id (FK)
├── water_source_id (FK)   ← created atomically on device creation
├── status (1=active)
└── version

water_sources
├── id (PK)
├── tenant_id
├── name                   ← display name on Water Page
├── source_type            ← inflow | outflow
├── measurement_type       ← METER_READING (Masgrau) | DAILY_CONSUMPTION (Shayp)
├── water_meter_id         ← used by Lynx (= zone_code)
└── is_legacy

connected_water_meter_hourly_records
├── connected_water_meter_device_id (FK)
├── date_time (unique per device)
├── consumption_value      ← raw reading from ETL
└── timestamps

lynx_club_configs
├── id (PK)
├── tenant_id (FK)
├── site_id (FK, nullable)
├── club_identifier        ← unique slug (e.g. "adare-manor")
├── api_key_hash           ← bcrypt of the API key
├── unit                   ← m3 | gallons
├── timezone
├── irrigation_day_start   ← e.g. "15:55"
├── last_sync_at
└── last_sync_status

lynx_water_records (staging)
├── lynx_club_config_id (FK)
├── zone_code              ← e.g. "1GR"
├── zone_name              ← e.g. "Hole 1 Green"
├── irrigation_date
├── total_volume / auto_volume / manual_volume
├── data_source            ← actual | scheduled | backfill
├── water_reading_id       ← NULL until promoted
└── UNIQUE(club_config_id, zone_code, irrigation_date)

water_readings (final destination for ALL types)
├── water_source_id (FK)
├── reading_value / consumption_value
├── reading_date
├── is_connected_device_record  ← true for Masgrau/Shayp
├── is_lynx_record              ← true for Lynx
└── measurement_type
```

---

## Step-by-Step: Adding a Masgrau or Shayp Device

### 1. Back Office: Create the device

Navigate to **Water Devices** in the sidebar and click **Add Water Device**.

| Field | What to enter |
|-------|---------------|
| Integration | `Masgrau (Modbus meter)` or `Shayp (API meter)` |
| Tenant | Search and select the tenant |
| Name | Display name for the Water Page (e.g. "PG1 - Nord") |
| Source Type | `inflow` or `outflow` |
| Device Reference ID | **Must match** the ID configured in the ETL agent (see step 2) |
| Site | The tenant's site this device belongs to |

**What happens on save** (atomic transaction in `ConnectedMeterDeviceAdminService.create()`):
1. Creates a `water_source` with `measurement_type` derived from integration type:
   - Masgrau → `METER_READING` (cumulative Modbus registers)
   - Shayp → `DAILY_CONSUMPTION` (API returns hourly deltas)
2. Creates a `connected_water_meter_device` linking to the water source
3. Creates a `water_source_default_association` linking source to site

### 2. ETL Agent: Configure the device reference

The ETL agent must send data tagged with the same `device_reference_id` you entered in step 1.

**Masgrau** (`maya-etl/pipelines/masgrau.py`):
- Config via env vars: `MASGRAU_CLIENTS=infinitum`
- Groups format: `PG1:Nord:100:masgrau-infinitum-pg1` ← last segment is the `device_reference_id`
- Agent reads Modbus registers and POSTs to `POST /api/v2/water/hourly-records`

**Shayp** (`maya-etl/pipelines/shayp.py`):
- Config: hardcoded in `SHAYP_DEVICES` list
- Each entry: `{"shayp_id": "N1C98504", "device_reference_id": "N5E4435C", ...}`
- Agent calls Shayp API, converts ml→litres, POSTs to `POST /api/v2/water/hourly-records`

### 3. Data Flow (automatic after setup)

```
ETL agent runs (cron / scheduler)
    ↓
POST /api/v2/water/hourly-records
  payload: { device_reference_id: "masgrau-infinitum-pg1", records: [{date_time, consumption_value}] }
    ↓
ConnectedWaterMeterService.resolveDeviceId()
  → SELECT id FROM connected_water_meter_devices WHERE device_reference_id = ?
    ↓
Upserts into connected_water_meter_hourly_records
    ↓
Every hour: `water:aggregate-connected-daily-readings` (routes/console.php)
    ↓
ConnectedWaterMeterDailyAggregationService:
  1. Finds all devices with hourly data
  2. For each device, determines start date (first missing daily reading)
  3. Branches on measurement_type:
     - METER_READING (Masgrau): uses last cumulative reading of the day
     - DAILY_CONSUMPTION (Shayp): sums all hourly values
  4. Calls WaterReadingService.createWaterReading(is_connected_device_record=true)
    ↓
water_readings row created → visible on Water Page
```

### 4. Verify

- Check `connected_water_meter_hourly_records` for incoming data
- Wait for the next hourly aggregation (or run manually: `php artisan water:aggregate-connected-daily-readings`)
- Check `water_readings` for the daily aggregated row
- Open the tenant's Water Page — device should appear as an outflow/inflow

---

## Step-by-Step: Adding a Lynx Club

### 1. Back Office: Create the Lynx device

Navigate to **Water Devices** → **Add Water Device**.

| Field | What to enter |
|-------|---------------|
| Integration | `Toro Lynx (Irrigation system)` |
| Tenant | Search and select the tenant (e.g. Adare Manor) |
| Name | Auto-filled from club_identifier |
| Source Type | Locked to `outflow` (Lynx zones are always outflows — D8) |
| Site | The tenant's site (optional, used for site consumption allocation) |
| Club Identifier | Unique slug (e.g. `adare-manor`) — must match agent config |
| Unit | `m3` or `gallons` (must match what the Lynx SQL Server uses) |
| Timezone | Club's timezone (e.g. `Europe/Dublin`) |
| Irrigation Day Start | When the irrigation day begins (e.g. `15:55`) |

**What happens on save** (`LynxClubConfigController.store()`):
1. Creates a `lynx_club_configs` row
2. Generates an API key (bcrypt hash stored, plaintext shown once)
3. Returns the API key — **copy it immediately, it won't be shown again**

### 2. Python Agent: Install and configure

Install the Lynx agent on a Windows machine that can reach the club's Lynx SQL Server.

**Config** (`config.yaml` or `.env`):
```yaml
LYNX_DB_HOST: <lynx-sql-server-ip>
LYNX_DB_PORT: 1433
LYNX_DB_NAME: lynx_main
LYNX_DB_USER: <read-only-user>
LYNX_DB_PASSWORD: <password>
LYNX_API_URL: https://api2.mayaglobal.io/api/v2/lynx/sync
LYNX_API_KEY: <the key from step 1>
LYNX_CLUB_IDENTIFIER: adare-manor
LYNX_IRRIGATION_DAY_START: "15:55"
LYNX_UNIT: gallons
```

**Schedule**: Windows Task Scheduler, daily at 06:00 local time.

### 3. Data Flow (automatic after setup)

```
Lynx agent runs daily at 06:00
    ↓
Connects to Lynx SQL Server (lynx_main database)
    ↓
Fetches from water_use_upload (actuals, 7-day window)
  + schedule_activity_download (fallback)
    ↓
Aggregates stations → zones (e.g. "1GR2" → zone "1GR")
  volume = (duration_seconds / 60) × station_flow
    ↓
Reconciles: actuals win over scheduled per zone/day
    ↓
POST /api/v2/lynx/sync (authenticated via X-Lynx-Api-Key header)
  payload: { club_identifier, records: [{zone_code, zone_name, irrigation_date, total_volume, ...}] }
    ↓
LynxIngestService.ingest():
  1. Upserts into lynx_water_records (UNIQUE on club+zone+date)
  2. Converts units if needed (gallons ↔ m3)
  3. Writes lynx_sync_logs
  4. Updates lynx_club_configs.last_sync_at
    ↓
Daily at 06:30: `water:promote-lynx-daily-readings`
    ↓
LynxDailyPromotionService.promoteAll():
  For each club config:
    For each unpromoted lynx_water_record (water_reading_id IS NULL):
      1. Find or auto-create water_source:
         - key: (tenant_id, water_meter_id = zone_code, is_legacy = false)
         - source_type = "outflow", measurement_type = DAILY_CONSUMPTION
      2. Create water_reading:
         - is_connected_device_record = true
         - is_lynx_record = true
         - reading_date = irrigation_date (already adjusted for day boundary)
      3. Create WaterSiteConsumption (if site linked)
      4. Set lynx_water_records.water_reading_id = new reading ID
    ↓
water_readings rows created → visible on Water Page
  - Lynx zones appear as outflows in all views
  - Hourly toggle hidden (Lynx is daily-only)
  - 🔗 icon in Water Records table
  - Calendar dots locked (can't toggle Lynx days)
```

### 4. Verify

- Run agent with `--dry-run` first to check reconciled output
- Check `lynx_sync_logs` in back office (Sync Logs button) for status
- Run `php artisan water:promote-lynx-daily-readings` manually or wait for 06:30
- Check `water_readings` for promoted rows with `is_lynx_record = true`
- Open tenant's Water Page — zones should appear as outflows

---

## Scheduled Commands (DevOps must confirm all are running)

| Command | Schedule | Purpose |
|---------|----------|---------|
| `water:aggregate-connected-daily-readings` | Hourly | Hourly records → daily `water_readings` (Masgrau + Shayp) |
| `water:promote-lynx-daily-readings` | Daily 06:30 | `lynx_water_records` → `water_readings` (Lynx) |
| `lynx:check-sync-health` | Daily 08:00 | Slack alert if club hasn't synced in >26h |

All declared in `core-2.0/routes/console.php`. DevOps must ensure `php artisan schedule:run` runs every minute.

---

## Troubleshooting

| Problem | Check |
|---------|-------|
| ETL pushes data but no hourly records appear | `device_reference_id` in agent config doesn't match the device in back office |
| Hourly records exist but no daily readings | Aggregation command not running, or device has < 24 hours of data (use `--force`) |
| Lynx sync returns 401 | API key mismatch — regenerate in back office and update agent config |
| Lynx sync succeeds but nothing on Water Page | Promotion command hasn't run yet — wait for 06:30 or run manually |
| Lynx zones not appearing as outflows | Water source auto-creation failed — check `water_sources` for `water_meter_id = zone_code` |
| "Device not found" on hourly-records POST | `connected_water_meter_devices` row doesn't exist or `device_reference_id` doesn't match |

---

## Current ETL Credential & Config Architecture

### How each agent currently gets its config

| Agent | Config source | Credential source | Discovery |
|-------|--------------|-------------------|-----------|
| **Masgrau** | SQLite credential store (`platform.db`) or `MASGRAU_*` env vars | Modbus IP/port in credential store, `MAYA_WATER_JWT` for Maya API | Manual: groups format `PG1:Nord:100:masgrau-infinitum-pg1` |
| **Shayp** | **Hardcoded in source code** (`SHAYP_DEVICES` list) | Bearer token in credential store or `SHAYP_BEARER_TOKEN` env var | None: must edit `.py` and redeploy |
| **Lynx** | `LYNX_*` env vars per agent instance | `LYNX_API_KEY` (from Maya back office), `LYNX_DB_*` for SQL Server | None: 1 agent = 1 club, reads all stations from SQL Server |

### ETL orchestrator (`maya-etl/`)

- **Flask web dashboard** on port 5001 (Docker: 3000) with HTMX UI
- **APScheduler** runs active pipelines on their `schedule_interval`
- **Credential store**: SQLite `platform.db` with Fernet-encrypted `fields_json`
  - Managed via `/credentials` web UI
  - Credential types: `modbus`, `bearer`, `api_key`, `api_key_secret`, `ftp`, `oauth2_refresh`, `basic_auth`, `hmac`
  - Lookup: `get_credentials(script_name, credential_type)` → DB first, env var fallback
- **Script registry**: `script` table tracks active/inactive pipelines, intervals, last run
- **Run history**: `run` table with PID, status, ok/fail/skipped counts

### The problem: no pull-from-Maya pattern

```
CURRENT (push-only):

  Back Office                    ETL Agent                    Maya API
  ┌──────────┐                  ┌──────────┐                ┌──────────┐
  │ Create   │   (manual)       │ Hardcoded│   POST data   │ Receives │
  │ device   │ ─── copy ID ──→ │ config   │ ────────────→ │ & stores │
  └──────────┘   to env/code    └──────────┘                └──────────┘

  Maya is receive-only. The ETL never asks "what devices should I sync?"
  Adding a device requires: back office + manual config change + redeploy.
```

### Proposed: pull-config-from-Maya pattern

```
PROPOSED (config-aware):

  Back Office                    Maya API                     ETL Agent
  ┌──────────┐   creates row   ┌──────────┐                ┌──────────┐
  │ Create   │ ──────────────→ │ Stores   │  GET /devices  │ Pulls    │
  │ device   │                 │ config   │ ←──────────── │ config   │
  └──────────┘                 └──────────┘  with creds    └──────────┘
                                     │                          │
                                     │       POST data          │
                                     │ ←────────────────────── │
                                     │                          │
```

**New API endpoint** (to build in `core-2.0`):
```
GET /api/v2/etl/water-devices?type=masgrau
GET /api/v2/etl/water-devices?type=shayp
GET /api/v2/etl/water-devices?type=lynx
```

Returns device configs the agent needs to know what to poll:

```json
// Masgrau/Shayp response
{
  "devices": [
    {
      "device_reference_id": "masgrau-infinitum-pg1",
      "tenant_id": "abc-123",
      "name": "PG1 - Nord",
      "integration_type": "masgrau",
      "status": "active",
      "config": {
        "host": "192.168.1.10",
        "port": 502,
        "register_base": 100
      }
    }
  ]
}

// Lynx response
{
  "clubs": [
    {
      "club_identifier": "adare-manor",
      "tenant_id": "def-456",
      "unit": "gallons",
      "timezone": "Europe/Dublin",
      "irrigation_day_start": "15:55",
      "api_key": "lynx_xxx..."  // only on first fetch or regenerate
    }
  ]
}
```

**Benefits:**
- Add a device in back office → ETL picks it up on next run (no redeploy)
- Single source of truth for device config (Maya DB, not scattered env vars)
- Shayp no longer hardcoded in source code
- Lynx: one centralized agent can serve multiple clubs
- Back office `status: inactive` → ETL skips device automatically

**Implementation steps (future work, not March scope):**

1. **Backend**: Add `GET /api/v2/etl/water-devices` endpoint with ETL-specific auth (service API key, not user JWT)
2. **Back office**: Add connection config fields per device type:
   - Masgrau: `host`, `port`, `register_base` (Modbus connection details)
   - Shayp: `shayp_meter_id`, `tz_offset` (Shayp API identifiers)
   - Lynx: already has all fields (club_identifier, unit, timezone, etc.)
3. **ETL agents**: Replace hardcoded config with `GET /api/v2/etl/water-devices?type=X` call at startup
4. **Credential bridge**: ETL-specific secrets (Modbus IPs, Shayp bearer tokens) could stay in credential store or move to Maya — depends on security requirements (on-prem Modbus IPs may not belong in cloud DB)
5. **Migration**: Keep env var fallback for existing deployments during transition

**What's already compatible today:**
- `device_reference_id` is the stable linking key — both back office and ETL use it
- Lynx `club_identifier` + `api_key` are already managed in Maya's back office
- The back office form we built stores all the fields needed for config-pull

---

## Key Source Files

| Component | Path |
|-----------|------|
| Masgrau ETL agent | `maya-etl/pipelines/masgrau.py` |
| Shayp ETL agent | `maya-etl/pipelines/shayp.py` |
| Lynx ETL agent | `maya-etl/pipelines/lynx/main.py` |
| Lynx aggregator | `maya-etl/pipelines/lynx/aggregator.py` |
| Lynx SQL queries | `maya-etl/pipelines/lynx/queries.py` |
| Hourly records API | `core-2.0/app/Http/Controllers/Water/ConnectedWaterMeterRecordController.php` |
| Device resolution | `core-2.0/app/Services/Water/ConnectedWaterMeterService.php` |
| Daily aggregation | `core-2.0/app/Services/Water/ConnectedWaterMeterDailyAggregationService.php` |
| Lynx ingest | `core-2.0/app/Services/Lynx/LynxIngestService.php` |
| Lynx promotion | `core-2.0/app/Services/Lynx/LynxDailyPromotionService.php` |
| Back office device admin | `core-2.0/app/Services/Water/ConnectedMeterDeviceAdminService.php` |
| Back office Lynx admin | `core-2.0/app/Http/Controllers/Lynx/LynxClubConfigController.php` |
| Schedule declarations | `core-2.0/routes/console.php` |
| Water API routes | `core-2.0/routes/api/water.php` |
| Back office page | `back-office/src/pages/water-devices/index.vue` |
| Back office store | `back-office/src/stores/waterDevice.ts` |
