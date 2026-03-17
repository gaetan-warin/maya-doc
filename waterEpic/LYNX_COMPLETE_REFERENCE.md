# Toro Lynx Integration — Complete Technical Reference

> Compiled from all project documentation. Single source of truth for Lynx implementation.

---

## 1. What Lynx Is

Toro Lynx is an on-premise irrigation control system at golf clubs. It runs on a local SQL Server (`lynx_main` database). Maya needs to pull daily water usage data from it into the Water Page.

**Pilot customer:** Adare Manor, County Limerick, Ireland.

---

## 2. Lynx SQL Server Database

### Connection

| Parameter | Value |
|-----------|-------|
| Engine | SQL Server |
| Database | `lynx_main` |
| Access | Read-only (SELECT only) |
| Connection | Remote to club's Lynx server (local network) |

### Key Concept: Irrigation Day vs Calendar Day

- **Adare Manor irrigation day start:** 3:55 PM (15:55)
- **Irrigation day span:** 3:55 PM Day 1 → 3:55 PM Day 2
- **Database stores data at midnight boundaries** — must account for offset

```python
from datetime import datetime, time, timedelta

def get_irrigation_day(timestamp: datetime, day_start: time) -> date:
    """If timestamp is before day_start, it belongs to the PREVIOUS irrigation day."""
    if timestamp.time() < day_start:
        return (timestamp - timedelta(days=1)).date()
    return timestamp.date()
```

### Table: `water_use_upload` — PRIMARY SOURCE (Satellite Actuals)

| Attribute | Value |
|-----------|-------|
| Granularity | Per station (most granular) |
| Time basis | Aligned with irrigation day |
| Retention | **7 days only** — then aggregated into `water_use` |
| Source | Satellite faceplate reports what actually ran |

**Key fields:**

| Field | Description |
|-------|-------------|
| `SUID` | System Unique ID — unique per station (stable) |
| `duration` | Runtime in seconds |
| `auto_duration` | Auto-scheduled irrigation duration |
| `total_duration` | Total duration (auto + manual) |
| `station_flow` | Flow rate for the station |

**Volume calculation:**

```sql
-- Manual watering
manual_duration = total_duration - auto_duration

-- Volume
total_gallons = (duration / 60) * station_flow

-- Join to get station location
SELECT wu.*, s.station_descriptor
FROM water_use_upload wu
JOIN station s ON wu.SUID = s.SUID
```

**Upload trigger rules:**
1. Irrigation must have stopped for ≥1 hour
2. Must hit an even hour (2:00, 4:00, 6:00, etc.)
3. Must occur before irrigation day ends
4. Last possible trigger: 2:00 PM (1-hour gap before 3:55 PM)

**Pros:** Most accurate — reports what faceplate actually ran. Station-level granularity. Captures both auto and manual watering.

**Cons:** 7-day retention. Depends on satellite communication. Overseed periods cause gaps.

### Table: `schedule_activity_download` — FALLBACK SOURCE (Confirmed Schedule)

| Attribute | Value |
|-----------|-------|
| Granularity | Per station |
| Time basis | Created when schedule sent to faceplate |
| Source | Confirmation of schedule delivery to satellite |
| Retention | Permanent |

**Key fields:**

| Field | Description |
|-------|-------------|
| `station_descriptor` | Station location tag (e.g., "1GR2") |
| `duration` | Scheduled runtime in seconds |
| `station_flow` | Flow rate |

**Pros:** Reliable, permanent retention, confirms schedule was received.

**Cons:** Doesn't reflect what actually happened. Can't capture rain holds/cancellations. Multiple records per day during overseed.

### Table: `water_use` — BACKFILL ONLY (Aggregated Daily Summary)

| Attribute | Value |
|-----------|-------|
| Granularity | Per hole, per zone, per day |
| Time basis | Midnight to midnight (NOT irrigation day) |
| Retention | Permanent |

**Verdict:** Only use for historical backfill. Cannot split by irrigation day. Loses station-level detail.

### Table: `schedule_activity` — DO NOT USE

Less reliable than `schedule_activity_download`. Skip entirely.

### Table: `station` — Station Lookup

- **Join on:** `SUID` (stable, unique per station)
- **Display:** `station_descriptor` (e.g., "1GR2") — can be renamed by customer

---

## 3. Station & Zone Naming Convention

**Format:** `1GR2` = Hole 1 / Green / Station 2

| Part | Meaning | Examples |
|------|---------|----------|
| `1` | Hole number | 1-18+ |
| `GR` | Area code | GR=Green, FW=Fairway, TE=Tee |
| `2` | Station number | 1, 2, 3... |

**Zone = hole + area code** (e.g., `1GR` = all stations in Hole 1 Green)

```python
import re

def parse_zone(station_descriptor: str) -> str:
    """1GR2 → 1GR, 12FW3 → 12FW"""
    match = re.match(r'^(\d+[A-Z]+)', station_descriptor)
    return match.group(1) if match else None
```

**Important:** Always join on `SUID`, use `station_descriptor` for display/grouping only.

---

## 4. Reconciliation Logic

```
For each zone, for each irrigation day:

1. CHECK water_use_upload (actuals)
   ├── Has data? → USE IT (data_source = "actual")
   │   - Sum station durations in zone
   │   - Volume = (duration/60) * station_flow
   │   - manual = total_duration - auto_duration
   │
   └── No data? → FALL BACK
       │
       2. CHECK schedule_activity_download
          ├── Has data? → USE IT (data_source = "scheduled")
          │   - Sum station durations in zone
          │   - All assumed auto (can't split)
          │
          └── No data? → NO RECORD (gap)
              - Don't invent data
```

### Edge Cases

| Scenario | Handling |
|----------|----------|
| Overseed (no actuals for days) | Fallback to scheduled data |
| Rain hold | Actuals show zero/reduced runtime — correct data |
| Multiple downloads/day (overseed) | Sum all download records for that station-day |
| Zero-duration actual | Include — means "scheduled but didn't run" |
| Station with no descriptor | Log warning, skip |
| Re-sync same day | Upsert — latest sync wins |
| Satellite failure | Accept gap — not Maya's responsibility |

---

## 5. Ingest API Contract (Laravel)

### Endpoint

```
POST /api/v2/lynx/sync
Header: X-Lynx-Api-Key: <per-club-key>
```

### Request

```json
{
  "sync_id": "uuid",
  "club_identifier": "adare-manor",
  "agent_version": "1.0.0",
  "sync_date": "2026-03-15",
  "unit": "m3",
  "records": [
    {
      "zone_code": "1GR",
      "zone_name": "Hole 1 Green",
      "irrigation_date": "2026-03-14",
      "total_volume": 45.2,
      "auto_volume": 42.1,
      "manual_volume": 3.1,
      "data_source": "actual",
      "station_count": 8
    }
  ]
}
```

**`data_source` enum:** `actual` | `scheduled` | `backfill`

### Response

```json
{
  "sync_id": "uuid",
  "status": "success",
  "accepted": 18,
  "rejected": 0,
  "errors": []
}
```

### Auth

- `X-Lynx-Api-Key` header
- Stored as bcrypt hash in `lynx_club_configs`
- Displayed once on generation, never again

---

## 6. Data Model (3 New Tables)

### `lynx_club_configs`

| Column | Type | Notes |
|--------|------|-------|
| id | bigint PK | |
| tenant_id | uuid FK | Links to Maya tenant |
| club_identifier | varchar unique | e.g., "adare-manor" |
| api_key_hash | varchar | bcrypt |
| unit | enum | "m3" or "gallons" |
| timezone | varchar | e.g., "Europe/Dublin" |
| irrigation_day_start | time | e.g., "15:55" |
| last_sync_at | timestamp nullable | |
| last_sync_status | enum nullable | success/partial/failed |
| created_at, updated_at | timestamps | |

### `lynx_water_records`

| Column | Type | Notes |
|--------|------|-------|
| id | bigint PK | |
| lynx_club_config_id | bigint FK | |
| zone_code | varchar | e.g., "1GR" |
| zone_name | varchar | e.g., "Hole 1 Green" |
| irrigation_date | date | |
| total_volume | decimal | |
| auto_volume | decimal | |
| manual_volume | decimal | |
| data_source | enum | actual/scheduled/backfill |
| station_count | int | |
| sync_id | uuid | For tracing |
| water_reading_id | bigint FK nullable | Links to water_readings if converted |
| created_at, updated_at | timestamps | |
| **UNIQUE** | (lynx_club_config_id, zone_code, irrigation_date) | Upsert on re-sync |

### `lynx_sync_logs`

| Column | Type | Notes |
|--------|------|-------|
| id | bigint PK | |
| lynx_club_config_id | bigint FK | |
| sync_id | uuid | |
| agent_version | varchar | |
| records_received | int | |
| records_accepted | int | |
| records_rejected | int | |
| status | enum | success/partial/failed |
| error_details | json nullable | Array of error objects |
| duration_ms | int | |
| created_at | timestamp | |

---

## 7. Python Sync Agent

### Tech Stack

| Component | Choice |
|-----------|--------|
| Language | Python 3.11+ |
| SQL Server | `pyodbc` or `pymssql` |
| HTTP | `requests` or `httpx` |
| Packaging | PyInstaller → single .exe |
| Config | `config.yaml` |
| Scheduling | Windows Task Scheduler (daily 06:00) |

### Config File

```yaml
lynx:
  host: "localhost"
  port: 1433
  database: "lynx_main"
  username: "maya_readonly"
  password: "****"

maya:
  api_url: "https://api2.mayaglobal.io"
  api_key: "club-specific-key"

site:
  irrigation_day_start: "15:55"
  timezone: "Europe/Dublin"
  unit: "m3"
```

### CLI

```bash
maya-lynx-agent.exe --config config.yaml              # Normal sync
maya-lynx-agent.exe --config config.yaml --dry-run     # Query only, don't push
maya-lynx-agent.exe --config config.yaml --backfill    # Include water_use history
maya-lynx-agent.exe --config config.yaml --verbose     # Debug logging
```

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Partial (some records rejected) |
| 2 | API unreachable |
| 3 | Lynx DB connection failure |
| 4 | Config error |

### Logging

- Rotating file: `maya-lynx-agent.log` (10MB x 5 rotations)
- Also stdout
- Every query logged with row count + execution time

### Daily Run Sequence

1. Connect to local Lynx SQL Server
2. Query `water_use_upload` (actuals, last 7 days)
3. Query `schedule_activity_download` (scheduled, same range)
4. Reconcile: actuals > scheduled > gap
5. Aggregate: station → zone (parse `station_descriptor`)
6. Handle irrigation day boundary (3:55 PM grouping)
7. Calculate volumes: `(duration_seconds / 60) * station_flow`
8. Push JSON to Maya API over HTTPS (port 443)
9. Log everything

---

## 8. Water Page Integration

| View | How Lynx Data Appears |
|------|----------------------|
| Water Usage card | Lynx zones in totals. Daily only (no hourly). |
| Water Usage modal | Zones in outflow selector. Hourly hidden for Lynx. |
| Days Watered calendar | Green dot if Lynx reported irrigation. Per-zone via outflow picker. |
| Water Budget | Lynx volume counts toward annual allowance. |
| Water Records table | "Connected Meter Logs" tab with link icon. |

**New capability:** Per-hole/zone breakdown when Lynx outflow selected.

**Zone → Outflow mapping:** Each zone_code maps to a `water_source` (outflow), auto-created on first sync.

---

## 9. Adare Manor Pilot Configuration

| Setting | Value |
|---------|-------|
| Location | Adare Manor, Co. Limerick, Ireland |
| Irrigation day start | 15:55 (3:55 PM) |
| Timezone | Europe/Dublin |
| Unit | m3 (TBC) |
| SQL Server access | Confirmed by MSM (Shaun Bowles, Feb 2026) |
| Status | Pilot, first implementation |

---

## 10. Known Limitations

| Limitation | Impact | Mitigation |
|-----------|--------|------------|
| 7-day retention on `water_use_upload` | Data lost if not captured | Daily sync (minimum weekly) |
| Overseed periods (~2x/year) | No actuals (continuous watering blocks trigger) | Auto-fallback to scheduled data |
| Satellite comms every 2h on even hours | Upload depends on communication | Accept gaps |
| Rain holds don't clear scheduled records | `schedule_activity_download` shows plan, not reality | Only `water_use_upload` reflects actual |
| Station tags can be renamed | Display names change | Join on SUID, display from descriptor |
| Last upload trigger: 2:00 PM | Late watering may not get uploaded | Accept limitation |
| Irrigation day ≠ calendar day | Data spans two dates | Group by irrigation day boundary |

---

## 11. Implementation Estimates

### Phase 1: Pilot (~16.5 dev days)

**A. Cloud API (Laravel) — 7.5 days**

| Task | Days |
|------|------|
| A1: API contract & migrations | 1 |
| A2: Club config & API key management | 1.5 |
| A3: Ingest endpoint | 1.5 |
| A4: Health monitoring | 1.5 |
| A5: Data access layer for Water Page | 1.5 |

**B. Python Agent — 6 days**

| Task | Days |
|------|------|
| B1: Lynx DB query layer | 1.5 |
| B2: Reconciliation engine | 2 |
| B3: HTTPS push to Maya | 1 |
| B4: CLI, scheduling, logging | 1 |
| B5: PyInstaller build | 0.5 |

**C. Integration — 3 days**

| Task | Days |
|------|------|
| C1: End-to-end integration test | 1.5 |
| C2: Adare Manor deployment | 1.5 |

### Phase 2: Productize (Post-Pilot)

- MSI installer with GUI wizard (2 days)
- Agent self-update (1.5 days)
- Fleet management dashboard (1.5 days)
- Zone mapping config UI (1 day)

---

## 12. Important: Meter Data vs Irrigation System Data

Connected meters (Shayp/Masgrau) measure **actual pump consumption**. Lynx measures **theoretical irrigation system output**. They are two views of the same water — **not additive**.

No client today has both. When one does:
- Do not sum them
- Use one source per outflow for budgets
- Delta between the two reveals leaks/inefficiency
- V2 feature — Q2 2026
