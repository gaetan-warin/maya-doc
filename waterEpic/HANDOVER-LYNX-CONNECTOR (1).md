# Toro Lynx Connector — Developer Handover Briefing

_Prepared: 2026-03-02 | Author: Val (via CTO assistant)_
_For: Water Page developer + AI code assistant_

---

## TL;DR

We need to sync water usage data from Toro's **Lynx** irrigation system (on-premise SQL Server at golf clubs) into Maya's Water Page. This is a **productized module** — same solution deployed at 100+ clubs. Adare Manor (Ireland) is the pilot. **Target: live by end of April 2026.**

You're the right person for this because you own the Water Page epic (253) and the connected meter / ETL integration layer. This briefing gives you everything you need to build both sides: the on-premise sync agent and the cloud ingest.

---

## What This Document Contains

1. [Business Context](#1-business-context)
2. [Architecture Decision](#2-architecture-decision)
3. [Lynx Database — Complete Technical Reference](#3-lynx-database)
4. [The Sync Agent (Python)](#4-sync-agent)
5. [The Cloud Ingest API (Laravel)](#5-cloud-ingest-api)
6. [Reconciliation Logic](#6-reconciliation-logic)
7. [Epic Breakdown — All Stories](#7-epic-breakdown)
8. [Build Order & Dependencies](#8-build-order)
9. [Critical Questions For You](#9-critical-questions)
10. [Reference Files in This Repo](#10-reference-files)
11. [Contacts](#11-contacts)

---

## 1. Business Context

- Toro Lynx is the most widely used irrigation control system on golf courses
- Maya customers want to see their Lynx irrigation data inside Maya's Water Insight page — automatically, daily, no manual input
- **Adare Manor** (Ireland) is our first customer for this. We've already confirmed access to their Lynx database via MSM (Toro's software/support partner)
- This is a commercial module — one-time installation fee per club. Every club with Lynx is a potential customer
- The user sees **one clean water usage number per zone per day**. They never see "scheduled vs actual" — that's our internal reconciliation logic

---

## 2. Architecture Decision

```
┌──────────────────────────┐         HTTPS (443)        ┌──────────────────────────┐
│   CLUB NETWORK            │  ─────────────────────►   │   MAYA CLOUD (AWS)        │
│                            │                           │                            │
│  ┌──────────────────────┐ │                           │  ┌──────────────────────┐ │
│  │ Lynx SQL Server      │ │                           │  │ Core 2.0 (Laravel 11)│ │
│  │ (lynx_main DB)       │ │                           │  │                      │ │
│  └──────────┬───────────┘ │                           │  │ POST /api/v2/lynx/   │ │
│             │ local SQL    │                           │  │   sync               │ │
│  ┌──────────▼───────────┐ │                           │  └──────────┬───────────┘ │
│  │ Maya Lynx Agent      │ │   JSON payload + API key  │             │              │
│  │ (Python .exe)        │─┼───────────────────────────┼─────────────┘              │
│  │ Runs daily 06:00     │ │                           │             │              │
│  └──────────────────────┘ │                           │  ┌──────────▼───────────┐ │
│                            │                           │  │ MySQL                │ │
└──────────────────────────┘                           │  │ (Water Page data)    │ │
                                                        │  └──────────┬───────────┘ │
                                                        │             │              │
                                                        │  ┌──────────▼───────────┐ │
                                                        │  │ Water Page 2.0       │ │
                                                        │  │ (Epic 253)           │ │
                                                        │  └──────────────────────┘ │
                                                        └──────────────────────────┘
```

**Why this approach:**
- Agent pushes outbound over HTTPS — **no firewall or VPN changes** needed at the club
- Same package for every club, config is the only variable
- Scales to 100+ clubs
- MSM tech or club IT installs once, runs automatically from then on

---

## 3. Lynx Database

### Connection
| Parameter | Value |
|-----------|-------|
| Engine | SQL Server |
| Database name | `lynx_main` |
| Access | Read-only (SELECT only) |
| Method | Remote connection to club's Lynx server (local network) |

> **Confidentiality:** MSM shared this DB structure under a cooperative agreement specifically for the Adare Manor project. This is not typically disclosed to third parties. Handle with discretion.

### Key Concept: Irrigation Day ≠ Calendar Day

Lynx uses an "irrigation day" that spans two calendar dates:

| Concept | Value (Adare Manor) |
|---------|---------------------|
| Irrigation day start | **3:55 PM** |
| Span | 3:55 PM Day 1 → 3:55 PM Day 2 |
| DB date boundary | Midnight |

This is configurable per site via `begin_water_daytime`. **Every query must account for this.**

### Tables

#### 3a. `water_use_upload` — PRIMARY SOURCE (Actual Data)

| Attribute | Detail |
|-----------|--------|
| Granularity | Per station (most granular) |
| Retention | **7 days only** — then rolls into `water_use` |
| Source | Satellite faceplate reports what actually ran |

**Key fields:**
| Field | Description |
|-------|-------------|
| `SUID` | System Unique ID — stable identifier per station |
| `duration` | Runtime in seconds |
| `auto_duration` | Auto-scheduled irrigation duration |
| `total_duration` | Total (auto + manual) duration |
| `station_flow` | Flow rate |

**Calculations:**
```sql
-- Volume
total_gallons = (duration / 60) * station_flow

-- Manual watering
manual_duration = total_duration - auto_duration

-- Join to get station location
SELECT wu.*, s.station_descriptor
FROM water_use_upload wu
JOIN station s ON wu.SUID = s.SUID
```

**Upload trigger rules (important for understanding data gaps):**
1. Irrigation must have stopped for ≥1 hour
2. Must hit an even hour (2:00, 4:00, 6:00...)
3. Must occur before irrigation day ends (3:55 PM for Adare)
4. Last possible trigger: 2:00 PM

**Known risk:** During **overseed periods** (~2x/year), continuous all-day watering prevents the trigger from ever firing. Data gaps are expected. This is a Lynx limitation.

#### 3b. `schedule_activity_download` — FALLBACK SOURCE (Scheduled Data)

| Attribute | Detail |
|-----------|--------|
| Granularity | Per station |
| Source | Confirmed schedule delivery to satellite faceplate |
| Retention | Permanent |

**Key fields:**
| Field | Description |
|-------|-------------|
| `station_descriptor` | Station location tag (e.g., "1GR2") |
| `duration` | Scheduled runtime in seconds |
| `station_flow` | Flow rate |

```sql
SELECT station_descriptor, duration, station_flow,
       (duration / 60.0) * station_flow AS total_gallons
FROM schedule_activity_download
WHERE [date_filters]
```

**Pros:** Reliable, always present, confirms schedule reached the faceplate.
**Cons:** Does NOT capture manual watering or rain holds. If rain held irrigation, this still shows the schedule as downloaded.

**Overseed note:** During overseed, schedules are chunked into batches → multiple download records per station per day. Sum them.

#### 3c. `water_use` — HISTORICAL BACKFILL ONLY

| Attribute | Detail |
|-----------|--------|
| Granularity | Per hole, per zone, per day |
| Time basis | Midnight to midnight (NOT irrigation day) |
| Retention | Permanent |

Only use for backfilling historical data. Loses granularity and can't align to irrigation day.

#### 3d. `schedule_activity` — DO NOT USE

Less reliable than `schedule_activity_download`. Skip it.

### Station Naming Convention

Format: `1GR2` = Hole 1 / GR (Green area) / Station 2

```
1GR2
│││└─ Station number
│└┘── Area code (GR = Green, FW = Fairway, etc.)
└──── Hole number
```

**Always join on `SUID`** for reliable identification. Parse `station_descriptor` for display/grouping only. Customers can rename station tags at any time.

**Zone aggregation:** Parse the area code (e.g., `1GR`) from `station_descriptor` to group stations into zones. `1GR` = all sprinklers on Hole 1 Green = one Maya "outflow."

---

## 4. Sync Agent (Python)

### What It Does (Each Daily Run)
1. Connects to local Lynx SQL Server (`lynx_main`)
2. Queries `water_use_upload` (actuals) — last 7 days
3. Queries `schedule_activity_download` (scheduled) — same range
4. **Reconciles**: uses actuals where available, fills gaps with scheduled (see Section 6)
5. **Aggregates**: station-level → zone-level (parses `station_descriptor` area codes)
6. **Handles irrigation day boundary** (groups by 3:55 PM spans, not midnight)
7. Detects/converts units (gallons ↔ m³)
8. Pushes one clean daily-per-zone dataset to Maya's ingest API over HTTPS
9. Reports sync status for health monitoring

### Tech Stack
| Component | Choice | Rationale |
|-----------|--------|-----------|
| Language | Python 3.11+ | Widely supported, good SQL Server libraries |
| SQL Server driver | `pyodbc` or `pymssql` | Standard for SQL Server from Python |
| HTTP client | `requests` or `httpx` | Push to Maya API |
| Packaging | PyInstaller → single .exe | No Python installation needed at club |
| Config | `config.yaml` | Simple for MSM tech to edit |
| Scheduling | Windows Task Scheduler | Native, no extra dependencies |

### Config File Structure
```yaml
# Maya Lynx Agent Configuration

lynx:
  host: "localhost"          # or IP of Lynx server
  port: 1433
  database: "lynx_main"
  username: "maya_readonly"
  password: "****"

maya:
  api_url: "https://api2.mayaglobal.io"
  api_key: "club-specific-key-from-backoffice"

site:
  irrigation_day_start: "15:55"   # 3:55 PM for Adare Manor
  timezone: "Europe/Dublin"
  unit: "m3"                      # or "gallons"
```

### CLI Interface
```bash
maya-lynx-agent.exe --config config.yaml              # Normal daily sync
maya-lynx-agent.exe --config config.yaml --dry-run     # Query + reconcile, don't push
maya-lynx-agent.exe --config config.yaml --backfill    # Include water_use table for history
maya-lynx-agent.exe --config config.yaml --verbose     # Debug logging
```

### Exit Codes
| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Partial success (some records rejected by API) |
| 2 | API unreachable / sync failure |
| 3 | Lynx DB connection failure |
| 4 | Config error |

### Logging
- Rotating file: `maya-lynx-agent.log` (10MB, 5 rotations)
- Also stdout
- Every query logged with row count + execution time

### Installation (Phase 1 — Manual)
1. Copy `C:\MayaLynx\` folder to Lynx server (or any machine on same network)
2. Edit `config.yaml` with Lynx DB creds + Maya API key
3. Run `maya-lynx-agent.exe --dry-run` to validate
4. Set up Windows Scheduled Task for daily 06:00 local
5. Verify first sync in Maya Back Office

---

## 5. Cloud Ingest API (Laravel / Core 2.0)

### Endpoint
```
POST /api/v2/lynx/sync
Header: X-Lynx-Api-Key: <per-club-key>
```

### Request Payload
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

`data_source` enum: `actual` (from `water_use_upload`), `scheduled` (from `schedule_activity_download`), `backfill` (from `water_use`)

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

### Data Storage

**KEY QUESTION FOR YOU:** Epic 253 designed the connected meter integration to be pluggable ("Shayp is first, Masgrau is next, no code changes when new sources are added"). **Can Lynx data land in the same connected meter tables?** If yes, this simplifies everything — Lynx becomes just another water data source feeding the same Water Page pipeline. If the existing schema doesn't fit (e.g., it's hourly device-level vs daily zone-level), we create dedicated tables:

- `lynx_club_configs` — per-club settings, API key hash, last sync timestamp
- `lynx_water_records` — zone-day records (the core data)
- `lynx_sync_logs` — audit trail, health monitoring

### Back Office Components (Vue 3)

1. **Lynx Connectors list** — all configured clubs, last sync, status (green/yellow/red)
2. **Club config form** — select tenant, set unit/timezone/irrigation day start, generate API key
3. **Sync log viewer** — per-club, last 30 syncs with status and error details
4. **Health alerts** — daily Laravel job flags clubs with no sync in >26h, sends to Slack

### Auth
- Custom middleware resolving `X-Lynx-Api-Key` to club_id
- Not using Passport/OAuth — the agent is a machine client, API key is simpler and appropriate
- Key stored as bcrypt hash, displayed once on generation

---

## 6. Reconciliation Logic

The user sees **one number per zone per day**. Behind the scenes:

```
For each zone, for each irrigation day:

1. CHECK water_use_upload (actuals)
   ├── Has data? → USE IT (mark as "actual")
   │   - Sum all station durations in this zone
   │   - Calculate volume: (duration/60) * station_flow
   │   - Split auto vs manual: manual = total_duration - auto_duration
   │
   └── No data? → FALL BACK
       │
       2. CHECK schedule_activity_download
          ├── Has data? → USE IT (mark as "scheduled")
          │   - Sum all station durations in this zone
          │   - Calculate volume same way
          │   - Cannot split auto vs manual (all assumed auto)
          │
          └── No data? → NO RECORD (gap)
              - Don't invent data
              - Health monitoring will flag the gap
```

### Station → Zone Aggregation
```python
# Parse zone from station_descriptor
# "1GR2" → zone = "1GR" (Hole 1 Green)
# "3FW1" → zone = "3FW" (Hole 3 Fairway)

import re

def parse_zone(station_descriptor: str) -> str:
    """Extract zone code from station descriptor.
    Format: {hole_number}{area_code}{station_number}
    Example: 1GR2 → 1GR, 12FW3 → 12FW
    """
    match = re.match(r'^(\d+[A-Z]+)', station_descriptor)
    return match.group(1) if match else None
```

### Irrigation Day Grouping
```python
from datetime import datetime, time, timedelta

def get_irrigation_day(timestamp: datetime, day_start: time) -> date:
    """Determine which irrigation day a timestamp belongs to.
    If timestamp is before day_start, it belongs to the PREVIOUS irrigation day.
    """
    if timestamp.time() < day_start:
        return (timestamp - timedelta(days=1)).date()
    return timestamp.date()
```

### Edge Cases
| Scenario | Handling |
|----------|----------|
| Overseed (no actuals for days) | Fallback to scheduled data automatically |
| Rain hold | Actuals show zero/reduced runtime — this IS the correct data |
| Multiple downloads/day (overseed) | Sum all download records for that station-day |
| Zero-duration actual record | Include it — means "scheduled but didn't run" (rain hold) |
| Station with no descriptor | Log warning, skip (don't crash) |
| Re-run for same day | Upsert — latest sync wins, old record soft-deleted |

---

## 7. Epic Breakdown — All Stories

### Phase 1: Adare Manor Pilot

| Story | Description | Effort | Depends On |
|-------|-------------|--------|------------|
| **A1** | API contract & data model (JSON schemas, DB migrations) | 1 day | — |
| **A2** | Club config & API key management (Back Office UI + middleware) | 1.5 days | A1 |
| **A3** | Ingest API endpoint (`POST /api/v2/lynx/sync`) | 1.5 days | A1, A2 |
| **A4** | Health monitoring (dashboard, alerts, sync logs) | 1.5 days | A2, A3 |
| **A5** | Data access layer for Water Page (endpoints feeding Epic 253) | 1.5 days | A1, A3 |
| **B1** | Agent: Lynx DB query layer (SQL Server connection, queries) | 1.5 days | A1 |
| **B2** | Agent: Reconciliation engine (actuals/scheduled merge, zone aggregation) | 2 days | B1 |
| **B3** | Agent: HTTPS push to Maya API (retry, error handling) | 1 day | B2, A3 |
| **B4** | Agent: CLI, scheduling, logging | 1 day | B1-B3 |
| **B5** | Agent: Windows installation package | 1 day | B4 |
| **C1** | End-to-end integration test (mock Lynx DB → API → Water Page) | 1.5 days | A3-A5, B4 |
| **C2** | Adare Manor pilot deployment + monitoring | 1.5 days | C1, B5 |

**Phase 1 total: ~16.5 days.** With agent + cloud in parallel: **~3.5 weeks calendar time.**

### Phase 2: Productize (Post-Pilot)

| Story | Description | Effort |
|-------|-------------|--------|
| **D1** | MSI installer with GUI config wizard | 2 days |
| **D2** | Agent self-update mechanism | 1.5 days |
| **D3** | Fleet management dashboard (100+ clubs view) | 1.5 days |
| **D4** | Zone mapping configuration UI (non-standard station naming) | 1 day |

---

## 8. Build Order

**Recommended sequence (single developer):**

```
Week 1:
  A1 → API contract & data model (DO THIS FIRST — unblocks everything)
  B1 → Agent DB queries (can start after A1 locks the payload shape)

Week 2:
  A2 + A3 → Club config + Ingest API
  B2 → Reconciliation engine

Week 3:
  A4 + A5 → Health monitoring + Data access
  B3 + B4 → Agent push + CLI/scheduling

Week 4:
  B5 → Agent install package
  C1 + C2 → Integration test + Pilot deploy
```

**If two devs are available:** Workstream A (cloud) and Workstream B (agent) are fully parallelizable after A1 is done.

---

## 9. Critical Questions For You

These are decisions only you can make because you know the Water Page codebase:

### Q1: Can Lynx data use Epic 253's connected meter tables?
Epic 253 created a pluggable connected meter integration (Shayp first). If that data model can accommodate daily zone-level data from Lynx, we skip building a separate data layer entirely. Lynx becomes just another source.
- If **yes** → A1 and A5 simplify dramatically
- If **no** → build the full `lynx_water_records` schema as described above

### Q2: What API shape does the Water Page frontend expect?
Story A5 needs to serve data in the exact format the Water Page Vue components consume. You already know this shape — define it in A1 so the agent can match it.

### Q3: Separate repo for the Python agent?
Recommendation: **yes** — `maya-lynx-agent`. Different language, different deployment target, different release cycle. But your call.

### Q4: How far back to backfill Adare Manor's historical data?
The `water_use` table has full history but uses midnight boundaries (not irrigation day). Suggestion: backfill last 12 months, accept the slight inaccuracy for historical data.

---

## 10. Reference Files in This Repo

All in `/projects/active/toro-lynx/`:

| File | What It Contains |
|------|-----------------|
| `technical-documentation.md` | Complete Lynx DB reference (tables, fields, queries, risks, architecture decision). **Read this first.** |
| `20260217-transcript-*.txt` | Full verbatim transcript of the MSM technical call. Useful if you need exact quotes or context on edge cases. |
| `epic253-waterpage2.0` | PDF export of Epic 253 (84 pages). The Water Page spec, backend tasks, QA status, and full GitLab comment thread with all decisions. |

---

## 11. Contacts

| Person | Role | When to Contact |
|--------|------|-----------------|
| **Val** | CEO/Founder | Architecture decisions, business questions, Adare Manor relationship |
| **Daniel Kaufmann** | MSM account contact | Arranging access, coordination |
| **Shaun Bowles** | MSM backend engineer (25 yrs exp) | Lynx DB questions, edge cases, station naming across clubs. Offered ongoing support — very helpful. Reach out via Daniel. |

---

## Quick Start for Your AI Code Assistant

If you're using Claude Code or similar to build this, here's the fastest path:

1. **Read `technical-documentation.md`** in this folder — it's the complete Lynx DB reference
2. **Answer Q1 above** (can Lynx use existing connected meter tables?) — this shapes everything
3. **Start with A1** — lock the API contract (JSON schemas for request/response)
4. **Build the Python agent (B1-B5) and Laravel API (A2-A5) in parallel**
5. **Test against Adare Manor's live Lynx DB** — we have confirmed access via MSM

The reconciliation logic in Section 6 has working Python pseudocode you can use directly.

---

_Questions? Reach out to Val. She has full context on the MSM relationship, Adare Manor expectations, and commercial model._
