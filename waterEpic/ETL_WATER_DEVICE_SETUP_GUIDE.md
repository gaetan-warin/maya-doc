# Water Device Setup Guide

> Internal documentation for the CS / DevOps team.
> Covers how to set up each water integration type end-to-end.

---

## Overview

Maya supports three types of water data integrations. Each follows the same pattern:

1. Get connection details from the site or provider
2. Configure the ETL agent to connect to the data source
3. Create the device in Maya Back Office so data lands in the right tenant
4. Verify data flows through to the Water Page

The **linking key** between the ETL and Back Office must match exactly — this is what connects incoming data to the correct customer.

---

## Prerequisites

Before setting up any water device, confirm:

- [ ] The tenant exists in Maya Back Office
- [ ] The tenant has at least one site configured
- [ ] Access to the ETL dashboard (`maya-etl` — typically port 5001)
- [ ] Access to Maya Back Office → **Water Devices** section in the sidebar

---

## Masgrau Setup (Modbus Water Meter)

### What is it?

Masgrau meters are physical Modbus/TCP devices installed on-site. They measure cumulative water consumption via Modbus registers. The ETL reads the registers every 15 minutes and pushes hourly readings to Maya.

### What you need from the site

| Item | Example | Who provides it |
|------|---------|----------------|
| Modbus gateway IP address | `192.168.1.50` | On-site technician |
| Modbus port | `502` (standard) | On-site technician |
| Number of pumps / pumping groups | 3 (PG1, PG2, PG3) | On-site technician |
| Register base address per pump | `100`, `200`, `300` | Masgrau documentation or technician |
| A label per pump | "Nord", "Centre", "Sud" | Site manager |

### Step 1 — Choose a device reference ID

Create a stable, unique identifier for each pump. Convention:

```
masgrau-{tenant-slug}-{pump-name}
```

Examples:
- `masgrau-infinitum-pg1`
- `masgrau-infinitum-pg2`
- `masgrau-infinitum-pg3`

> This ID is the link between ETL and Back Office. It cannot change once data starts flowing.

### Step 2 — Configure the ETL credential store

1. Open the ETL dashboard → **Credentials** → **Create**
2. Fill in:

| Field | Value |
|-------|-------|
| Name | `{Tenant name} Modbus` (e.g. "Infinitum Living Modbus") |
| Type | `modbus` |
| Host | The Modbus gateway IP |
| Port | `502` |
| Timeout | `10` |
| Groups | See format below |

**Groups format** (comma-separated, one per pump):

```
{name}:{label}:{register_base}:{device_reference_id}
```

Example for 3 pumps:

```
PG1:Nord:100:masgrau-infinitum-pg1,PG2:Centre:200:masgrau-infinitum-pg2,PG3:Sud:300:masgrau-infinitum-pg3
```

3. **Link** the credential to the `masgrau.py` script

### Step 3 — Create devices in Maya Back Office

Go to **Water Devices** → **Add Water Device**. Create one entry per pump:

| Field | Value |
|-------|-------|
| Integration | Masgrau (Modbus meter) |
| Tenant | Select the tenant |
| Name | Display name (e.g. "PG1 - Nord") |
| Source Type | Usually `outflow` |
| Device Reference ID | `masgrau-infinitum-pg1` — **must match ETL groups** |
| Site | Select the tenant's site |

Repeat for each pump.

### Step 4 — Verify

1. Check the ETL dashboard → **Runs** → `masgrau` should show `status: success`
2. Wait ~1 hour for the daily aggregation cron (`water:aggregate-connected-daily-readings`)
3. Open the tenant's **Water Page** → the pump should appear as an outflow with data

### How data flows

```
Modbus gateway → ETL reads registers every 15min
  → POST /api/v2/water/hourly-records (matched by device_reference_id)
  → Stored in connected_water_meter_hourly_records
  → Hourly cron aggregates into water_readings (measurement_type: METER_READING)
  → Visible on Water Page
```

---

## Shayp Setup (API Water Meter)

### What is it?

Shayp meters are IoT water meters that report to Shayp's cloud API. The ETL queries Shayp's API to pull hourly consumption data, then pushes it to Maya.

### What you need from the provider

| Item | Example | Who provides it |
|------|---------|----------------|
| Shayp meter ID | `N1C98504` | Shayp (on the device or their dashboard) |
| Device reference ID | `N5E4435C` | Shayp (unique per physical meter) |
| Timezone offset | `2` (CET+1) | Determine from site location |
| Shayp API bearer token | `eyJ...` | Shayp account manager |

### Step 1 — Ensure API credentials exist

1. Open the ETL dashboard → **Credentials**
2. Check if a `bearer` credential for Shayp already exists
3. If not, create one:

| Field | Value |
|-------|-------|
| Name | `Shayp API` |
| Type | `bearer` |
| Bearer Token | The token provided by Shayp |

4. Link to `shayp.py` script

> One bearer token works for all Shayp meters. You only need to do this once.

### Step 2 — Add the device to the ETL agent

> ⚠️ **Current limitation:** Shayp devices are hardcoded in source code. This requires a code change and redeployment.

Edit `maya-etl/pipelines/shayp.py` → find the `SHAYP_DEVICES` list and add an entry:

```python
SHAYP_DEVICES = [
    # ... existing devices ...
    {
        "shayp_id": "N1C98504",           # Shayp's internal meter ID
        "device_reference_id": "N5E4435C", # Unique reference for Maya
        "tz_offset": 2,                    # Timezone offset from UTC
        "name": "Naxhelet",               # Human-readable label
    },
]
```

Commit and redeploy the ETL.

### Step 3 — Create the device in Maya Back Office

Go to **Water Devices** → **Add Water Device**:

| Field | Value |
|-------|-------|
| Integration | Shayp (API meter) |
| Tenant | Select the tenant |
| Name | Display name (e.g. "Naxhelet") |
| Source Type | Usually `outflow` |
| Device Reference ID | `N5E4435C` — **must match the ETL code** |
| Site | Select the tenant's site |

### Step 4 — Verify

1. Check ETL dashboard → **Runs** → `shayp` should show records pushed
2. Wait ~1 hour for aggregation
3. Open tenant's **Water Page** → meter should appear as an outflow

### How data flows

```
Shayp cloud → ETL queries Shayp API every 15min
  → Converts ml → litres, adjusts timezone
  → POST /api/v2/water/hourly-records (matched by device_reference_id)
  → Stored in connected_water_meter_hourly_records
  → Hourly cron aggregates into water_readings (measurement_type: DAILY_CONSUMPTION)
  → Visible on Water Page
```

---

## Lynx Setup (Toro Irrigation System)

### What is it?

Toro Lynx is an on-premise irrigation control system used at golf clubs. It runs on a local SQL Server. A Python agent installed at the club reads irrigation data daily and pushes it to Maya.

Lynx provides per-zone data (e.g. "Hole 1 Green", "Hole 3 Fairway") — granularity that manual entry or meters don't have.

### What you need from the club

| Item | Example | Who provides it |
|------|---------|----------------|
| Lynx SQL Server IP | `10.0.2.100` | Club IT or MSM (Shaun Bowles) |
| SQL Server port | `1433` (standard) | Club IT |
| Read-only DB username | `maya_reader` | Club IT (create a dedicated user) |
| DB password | `***` | Club IT |
| Database name | `lynx_main` | Usually standard |
| Irrigation day start time | `15:55` | Course superintendent / MSM |
| Volume unit | `gallons` or `m3` | Course superintendent |
| Club timezone | `Europe/Dublin` | Known from location |

### Step 1 — Create the club in Maya Back Office

Go to **Water Devices** → **Add Water Device**:

| Field | Value |
|-------|-------|
| Integration | Toro Lynx (Irrigation system) |
| Tenant | Select the tenant (e.g. Adare Manor) |
| Name | Club name |
| Site | Select the tenant's site |
| Club Identifier | Unique slug: `adare-manor` |
| Unit | `gallons` or `m3` |
| Timezone | e.g. `Europe/Dublin` |
| Irrigation Day Start | e.g. `15:55` |

On save, an **API key** is generated and displayed once. **Copy it immediately.**

> The API key authenticates the agent when it pushes data to Maya. If lost, you can regenerate it from the device list (key icon).

### Step 2 — Install the Lynx agent on-site

Requirements:
- Windows machine at the club that can reach the Lynx SQL Server
- Outbound HTTPS (port 443) to `api2.mayaglobal.io`
- Python 3.11+ or the packaged `.exe`

**Option A — Packaged .exe (recommended for production):**

1. Copy `MayaLynxAgent.exe` to `C:\MayaLynx\`
2. Create `C:\MayaLynx\.env`:

```ini
LYNX_DB_HOST=10.0.2.100
LYNX_DB_PORT=1433
LYNX_DB_NAME=lynx_main
LYNX_DB_USER=maya_reader
LYNX_DB_PASSWORD=<password from club IT>

LYNX_API_URL=https://api2.mayaglobal.io/api/v2/lynx/sync
LYNX_API_KEY=<API key from Back Office step 1>
LYNX_CLUB_IDENTIFIER=adare-manor

LYNX_UNIT=gallons
LYNX_IRRIGATION_DAY_START=15:55
```

3. Schedule in **Windows Task Scheduler**:
   - Program: `C:\MayaLynx\MayaLynxAgent.exe`
   - Trigger: Daily at **06:00** local time
   - Run whether user is logged on or not

**Option B — Python directly (for testing):**

```bash
cd maya-etl/pipelines/lynx
pip install -r requirements.txt
python main.py --config /path/to/.env
```

### Step 3 — Test with dry run

Before going live, run with `--dry-run`:

```bash
MayaLynxAgent.exe --dry-run
```

This queries the SQL Server, reconciles data, and prints the payload JSON **without pushing to Maya**. Review the output with MSM to confirm:

- Zone codes match expected holes (e.g. `1GR` = Hole 1 Green)
- Volumes look reasonable
- Irrigation day boundaries are correct

### Step 4 — Go live and verify

1. Run the agent once without `--dry-run`
2. In Back Office → Water Devices → click **Sync Logs** (list icon) on the Lynx device
3. Confirm: `status: success`, `accepted > 0`, `rejected: 0`
4. Wait for daily promotion at 06:30 (or run manually: `php artisan water:promote-lynx-daily-readings`)
5. Open tenant's **Water Page**:
   - Lynx zones appear as outflows in Water Usage card
   - Days Watered calendar shows green dots for irrigated days
   - Water Records table shows entries with 🔗 icon

### Step 5 — Optional: backfill historical data

To load historical data from the Lynx `water_use` table:

```bash
MayaLynxAgent.exe --backfill --months 12
```

This pulls up to 12 months of historical irrigation data. Run once after initial setup.

### How data flows

```
Lynx SQL Server → Agent runs daily at 06:00
  → Queries water_use_upload (actuals, 7-day window)
  → Queries schedule_activity_download (fallback)
  → Reconciles: actuals win per zone/day
  → Aggregates stations → zones (e.g. "1GR2" → zone "1GR")
  → POST /api/v2/lynx/sync (authenticated with API key)
  → Stored in lynx_water_records (staging table)
  → Daily 06:30 cron promotes to water_readings (is_lynx_record = true)
  → Visible on Water Page as zone-level outflows
```

### Known Lynx behaviors

| Behavior | Impact | Handling |
|----------|--------|----------|
| **Overseed periods** (~2x/year) | Continuous watering prevents satellite upload trigger → no actuals | Agent falls back to scheduled data automatically |
| **7-day retention** on `water_use_upload` | Old actuals are lost | Agent must run at least weekly (daily recommended) |
| **Satellite comms** every 2h on even hours | Data arrives in batches | Agent runs at 06:00 to catch overnight uploads |
| **Rain holds** | Schedule shows irrigation but actual runtime was zero | `water_use_upload` reflects the real zero — agent uses it |
| **Station renames** | Tags can change in Lynx | Agent joins on SUID (stable), not station name |

---

## Quick Reference: Matching Keys

The single most important thing: **the ETL identifier must match the Back Office identifier.**

| Integration | ETL side | Back Office side | Must match |
|---|---|---|---|
| Masgrau | `groups` field: `...:{device_reference_id}` | Device Reference ID field | Exactly |
| Shayp | `SHAYP_DEVICES` list: `device_reference_id` | Device Reference ID field | Exactly |
| Lynx | `.env`: `LYNX_CLUB_IDENTIFIER` | Club Identifier field | Exactly |
| Lynx | `.env`: `LYNX_API_KEY` | Generated API key | Copy from Back Office |

---

## Troubleshooting

### Masgrau

| Symptom | Cause | Fix |
|---------|-------|-----|
| ETL run shows `Modbus read failed` | Wrong IP, port, or gateway offline | Check IP from credential store, ping the gateway |
| ETL succeeds but no data on Water Page | `device_reference_id` mismatch | Compare ETL groups string with Back Office device |
| Hourly records exist but no daily readings | Aggregation cron not running | Ask DevOps to verify `water:aggregate-connected-daily-readings` runs hourly |

### Shayp

| Symptom | Cause | Fix |
|---------|-------|-----|
| ETL shows `401 Unauthorized` | Shayp bearer token expired | Get new token from Shayp, update credential store |
| ETL succeeds but no data on Water Page | `device_reference_id` mismatch | Compare code list with Back Office device |
| Adding a new meter requires redeployment | Devices are hardcoded | Edit `shayp.py`, commit, redeploy |

### Lynx

| Symptom | Cause | Fix |
|---------|-------|-----|
| Agent exits with code 3 | Cannot connect to SQL Server | Check IP, port, credentials, firewall |
| Agent exits with code 2 | Maya API rejected the push | Check API key matches Back Office, check `club_identifier` |
| Agent exits with code 4 | Missing config | Check all required env vars are set |
| `--dry-run` works but live push fails | Network issue | Ensure outbound HTTPS (443) to `api2.mayaglobal.io` |
| Sync log shows `status: success` but nothing on Water Page | Promotion hasn't run | Wait for 06:30 or run `php artisan water:promote-lynx-daily-readings` |
| Zones show wrong names | Station descriptors changed in Lynx | Agent auto-derives names from zone codes — names update on next sync |

---

## Contacts

| Role | Who | When to contact |
|------|-----|----------------|
| MSM / Lynx expert | Shaun Bowles | Lynx SQL Server access, irrigation day config, zone validation |
| Shayp account | (TBD) | Bearer token renewal, meter provisioning |
| DevOps | (TBD) | ETL deployment, cron verification, credential store access |
| Maya Back Office | Admin team | Device creation, tenant/site setup |
