# Water Epic — Comprehensive Gap Analysis Report

**Date:** 2026-03-10 (updated to align with Epic 341)
**Scope:** Core 2.0 (Laravel backend) + Web (Vue 3 frontend)
**Source documents:** Status_epic.md, technical-documentation.md, HANDOVER-LYNX-CONNECTOR.md, MSM call transcript, Epic 341 (GitLab — replaces Epic 253)

---

## Scope Change Notice

**Epic 341 replaces Epic 253.** The following features are **cancelled** and no longer represent gaps:
- Irrigation notifications & planning (entire module)
- Mobile push notifications & mobile APIs
- 4-color calendar system (simplified to 2 colors)
- Notification-related settings (frequency, mobile toggle, reminder time)
- Daily irrigation logs table

This gap analysis reflects only what is needed for **Epic 341 delivery** (end of March 2026).

---

## Executive Summary

| Capability Area | Backend | Frontend | Overall | Action |
|----------------|---------|----------|---------|--------|
| Water Sources & Readings (CRUD) | ~90% | ~85% | ~87% | Verify + fix 4 bugs |
| Dashboard & Analytics (Cards, Charts) | ~80% | ~80% | ~80% | Verify all cards/modals |
| Days Watered Calendar | ~70% | ~60% | ~65% | Simplify to 2 colors, remove notification logic |
| Connected Meters (Shayp/Masgrau) | ~85% | ~75% | ~80% | Verify + promote staging items |
| Toro Lynx Connector | 0% | 0% | 0% | **BUILD — primary deliverable** |
| ~~Irrigation Notifications~~ | — | — | — | ~~CANCELLED~~ |
| ~~Mobile APIs~~ | — | — | — | ~~CANCELLED~~ |

**Critical finding:** Water API routes confirmed registered via `RouteServiceProvider.boot()` under `api/v2`.

---

## 1. WHAT IS FULLY IMPLEMENTED

### 1.1 Water Sources Management
| Component | Status | Location |
|-----------|--------|----------|
| WaterSource model (inflow/outflow) | DONE | `core-2.0/app/Models/WaterSource.php` |
| WaterSourceType model | DONE | `core-2.0/app/Models/WaterSourceType.php` |
| WaterSourceDefaultAssociation model | DONE | `core-2.0/app/Models/WaterSourceDefaultAssociation.php` |
| Source CRUD API endpoints | DONE | `GET/POST/PUT/DELETE /settings/water/sources` |
| Source service layer | DONE | `core-2.0/app/Services/Water/WaterSourceService.php` |
| Source repository + interface | DONE | `core-2.0/app/Repositories/Water/WaterSourceRepository.php` |
| Settings UI (Inflow form) | DONE | `web/src/components/setting/water/WaterInflow.vue` |
| Settings UI (Outflow form) | DONE | `web/src/components/setting/water/WaterOutflow.vue` |
| Settings UI (main index) | DONE | `web/src/components/setting/water/Index.vue` |
| Enums (SourceType, MeasurementType, MeterUnit) | DONE | `core-2.0/app/Enums/Water*.php` |

### 1.2 Water Readings
| Component | Status | Location |
|-----------|--------|----------|
| WaterReading model | DONE | `core-2.0/app/Models/WaterReading.php` |
| WaterSiteConsumption model | DONE | `core-2.0/app/Models/WaterSiteConsumption.php` |
| Reading CRUD API endpoints | DONE | `GET/POST/PUT/DELETE /water/readings` |
| Reading service layer | DONE | `core-2.0/app/Services/Water/WaterReadingService.php` |
| Event-driven consumption distribution | DONE | Events: `WaterReadingCreated/Updated/Deleted` + Listeners |
| Request validation classes (14 total) | DONE | `core-2.0/app/Http/Requests/Water/` |
| API response formatters | DONE | `WaterReadingResource.php`, `WaterSourceListResource.php` |

### 1.3 Dashboard & Analytics
| Component | Status | Location |
|-----------|--------|----------|
| Dashboard API | DONE | `GET /water/dashboard` |
| Statistics API | DONE | `GET /water/statistics` |
| Usage graph API | DONE | `GET /water/usage` |
| Budget graph API | DONE | `GET /water/budget` |
| ET data API | DONE | `GET /water/et` |
| Rainfall data API | DONE | `GET /water/rainfall` |
| Calendar data API | DONE | `GET /water/calendar` |
| Dashboard service | DONE | `core-2.0/app/Services/Water/WaterDashboardService.php` |
| Usage service | DONE | `core-2.0/app/Services/Water/WaterUsageService.php` |
| Main water view | DONE | `web/src/views/water.vue` |
| Header section | DONE | `web/src/components/water2/sections/WaterHeaderSection.vue` |
| Top section (6 cards) | DONE | `web/src/components/water2/sections/WaterTopSection.vue` |
| ET Card | DONE | `web/src/components/water2/components/cards/ETCard.vue` |
| Rainfall Card | DONE | `web/src/components/water2/components/cards/RainfallCard.vue` |
| Water Usage Card | DONE | `web/src/components/water2/components/cards/WaterUsageCard.vue` |
| Days Watered Card | DONE | `web/src/components/water2/components/cards/DaysWateredCard.vue` |
| Site Conditions Card | DONE | `web/src/components/water2/components/cards/SiteConditionsCard.vue` |
| Water Budget Card | DONE | `web/src/components/water2/components/cards/WaterBudgetCard.vue` |
| Water Usage Modal (charts) | DONE | `web/src/components/water2/components/modals/WaterUsageModalContent.vue` |
| ET History Modal | DONE | `web/src/components/water2/components/modals/ETModalContent.vue` |
| Rainfall History Modal | DONE | `web/src/components/water2/components/modals/RainfallModalContent.vue` |
| Budget Modal | DONE | `web/src/components/water2/components/modals/WaterBudgetModalContent.vue` |
| Data mapper utilities | DONE | `web/src/components/water2/utils/dashboardDataMapper.js` |
| Unit conversion (metric/imperial) | DONE | `web/src/composables/conversions/units/water.js` |
| Pinia store (27+ actions) | DONE | `web/src/store/water.js` |

### 1.4 Tenant & Source Water Settings
| Component | Status | Location |
|-----------|--------|----------|
| TenantWaterSettings model | DONE | `core-2.0/app/Models/TenantWaterSettings.php` |
| SourceWaterSettings model | DONE | `core-2.0/app/Models/SourceWaterSettings.php` |
| Tenant settings API | DONE | `GET/PUT /settings/water/tenant/{tenant}` |
| Source settings API | DONE | `PUT /settings/water/source/{source}` |
| Period validation API | DONE | `POST /settings/water/source/{source}/validate-period` |
| Water Configuration UI | DONE | `web/src/components/setting/water/components/WaterConfiguration.vue` |
| ET Factor setting | DONE | In WaterConfiguration.vue |

### 1.5 Connected Water Meter (Basic Model)
| Component | Status | Location |
|-----------|--------|----------|
| ConnectedWaterMeterDevice model | DONE | `core-2.0/app/Models/ConnectedWaterMeterDevice.php` |
| ConnectedWaterMeterHourlyRecord model | DONE | `core-2.0/app/Models/ConnectedWaterMeterHourlyRecord.php` |
| Hourly record ingestion API | DONE | `POST /water/hourly-records` |
| Daily aggregation service | DONE | `core-2.0/app/Services/Water/ConnectedWaterMeterDailyAggregationService.php` |
| Aggregation command | DONE | `core-2.0/app/Console/Commands/Water/AggregateConnectedWaterMeterDailyReadings.php` |

### 1.6 Database Migrations
All **21 water-related migrations** are present, covering:
- Water sources, readings, site consumption, associations
- Connected meter devices and hourly records
- Tenant and source water settings
- Legacy data archiving and migration
- Precision improvements and field additions

### 1.7 Data Migration Commands
**10 console commands** for migrating legacy water data are implemented.

---

## 2. GAPS REQUIRING ACTION (aligned with Epic 341)

### 2.1 Water Record Bugs — Must Fix

**Severity: HIGH**

| Bug # | Description | Status | Action |
|-------|-------------|--------|--------|
| #2254 | Cannot delete water records | Bug | **FIX** |
| #2258 | Cannot update water records | Bug | **FIX** |
| #2263 | Previous reading value wrong in message | Bug | **FIX** |
| #2283 | Calendar total consumption wrongly updated | Bug | **FIX** |
| #2259 | Calendar error on status change | Bug | **CLOSE** — notification feature cancelled |
| #2268 | Irrigation notifications not generating | Bug | **CLOSE** — notification system cancelled |

### 2.2 Days Watered Calendar — Needs Simplification

**Severity: MEDIUM**

| Component | Current Status | Required by Epic 341 |
|-----------|---------------|---------------------|
| Calendar UI (month view) | DONE (4-color) | Simplify to 2 colors (green/red) |
| Outflow filter dropdown | DONE | Keep |
| Status color legend | DONE (green/red/blue/light-blue) | Reduce to green/red only |
| Side panel for status selection | DONE (`OutflowStatusPanel.vue`) | Simplify — toggle green ↔ red only |
| Click-to-toggle | TODO (`useDateClickHandler.js:46`) | Implement with connected meter/Lynx data protection |
| Light blue "planned" dots | DONE | **REMOVE** |
| Calendar card logic (4 colors) | TODO (#1230) | **CANCEL** — 2 colors only |

### 2.3 Notification UI — Must Remove

**Severity: HIGH (dead code calling non-existent endpoints)**

The frontend has a complete notification UI that must be **removed** per Epic 341:

| Component | Current Status | Action |
|-----------|---------------|--------|
| Notification list UI (past 7 days) | DONE | **REMOVE from UI** |
| Multi-select & select all controls | DONE | **REMOVE** |
| Bulk mark Yes/No controls | DONE | **REMOVE** |
| Single notification expansion + input | DONE | **REMOVE** |
| NotificationCard.vue | DONE | **REMOVE** |
| BulkConsumptionPanel.vue | DONE | **REMOVE** |
| FE store: `fetchIrrigationNotifications()` | DONE | **DISABLE** |
| FE store: `updateIrrigationNotification()` | DONE | **DISABLE** |
| FE store: `batchUpdateIrrigationNotifications()` | DONE | **DISABLE** |

### 2.4 Settings — Remove Notification-Related Fields

**Severity: LOW**

| Setting | Current Status | Action |
|---------|---------------|--------|
| Annual water allowance | DONE | Keep |
| Irrigation period (from/to) | DONE | Keep |
| ET correction factor | DONE | Keep |
| Irrigation frequency | DONE in UI | **REMOVE from UI** (keep code) |
| Mobile notifications toggle | DONE in UI | **REMOVE from UI** |
| Preferred reminder time | DONE in UI | **REMOVE from UI** |

### 2.5 Connected Meter — Needs Verification & Promotion

**Severity: MEDIUM**

| Component | Status | Action |
|-----------|--------|--------|
| IoT vs Manual distinction (#384) | IN REVIEW | **VERIFY** and merge |
| Bulk value ingestion (#438) | TESTING | **VERIFY** and merge |
| Calendar status update (#440) | TESTING | **VERIFY** and merge |
| 8 staging items (#376–#383) | ON STAGING | **VERIFY** and promote to production |

### 2.6 Graph-Data Endpoint — Missing

**Severity: LOW**

FE store calls `GET /water/graph-data` (water.js:118) but endpoint doesn't exist. Either create in `WaterController` or redirect FE to existing `/water/usage`.

---

## 3. WHAT MUST BE BUILT (Primary Deliverable)

### 3.1 Toro Lynx Connector — Entire Module

**Severity: CRITICAL (committed to Adare Manor, target end of March 2026)**

**Zero code exists in the codebase.**

#### 3.1a Cloud Ingest API (Laravel side) — NOT STARTED
| Story | Description | Status |
|-------|-------------|--------|
| A1 | API contract & data model (JSON schemas, DB migrations) | NOT STARTED |
| A2 | Club config & API key management (Back Office UI + middleware) | NOT STARTED |
| A3 | Ingest API endpoint (`POST /api/v2/lynx/sync`) | NOT STARTED |
| A4 | Health monitoring — minimal MVP (Slack alerts for CS team) | NOT STARTED |
| A5 | Data access layer for Water Page (zones as outflows) | NOT STARTED |

**Required database tables (none exist):**
- `lynx_club_configs` — per-club settings, API key hash, last sync timestamp
- `lynx_water_records` — zone-day records
- `lynx_sync_logs` — audit trail, health monitoring

**Required backend components (none exist):**
- LynxSyncController
- X-Lynx-Api-Key authentication middleware
- LynxClubConfig model
- LynxWaterRecord model
- LynxSyncLog model
- Health monitoring daily job (alert on >26h without sync)
- Slack notification integration for missed syncs

#### 3.1b Sync Agent (Python) — NOT STARTED
| Story | Description | Status |
|-------|-------------|--------|
| B1 | Agent: Lynx DB query layer (SQL Server connection) | NOT STARTED |
| B2 | Agent: Reconciliation engine (actuals/scheduled merge) | NOT STARTED |
| B3 | Agent: HTTPS push to Maya API | NOT STARTED |
| B4 | Agent: CLI, scheduling, logging | NOT STARTED |
| B5 | Agent: Windows package (.exe via PyInstaller) | NOT STARTED |

**Required Python components (none exist):**
- SQL Server connection to `lynx_main` via `pyodbc`/`pymssql`
- Queries against `water_use_upload` (actuals, 7-day retention)
- Queries against `schedule_activity_download` (scheduled, fallback)
- Reconciliation logic (actuals preferred, scheduled as fallback)
- Station → Zone aggregation (parse `station_descriptor`)
- Irrigation day boundary handling (3:55 PM → 3:55 PM for Adare)
- Unit detection and normalization (gallons ↔ m³)
- HTTPS push with retry and error handling
- `config.yaml` loader
- PyInstaller packaging to `.exe`
- Windows Task Scheduler integration

#### 3.1c Back Office UI (Vue) — NOT STARTED (Minimal MVP)
- Club config form (tenant, units, timezone, irrigation day start, API key generation)
- Basic sync status visibility for CS team
- Full dashboard and sync log viewer deferred to V2

#### 3.1d Water Page Integration — NOT STARTED
- Lynx zones appear as outflows in all views (Usage card, Usage modal, Days Watered, Budget, Records)
- Per-hole/zone breakdown when Lynx outflow is selected
- Source icon: 🔗 irrigation system
- "Hourly" toggle hidden for Lynx outflows (daily data only)

---

## 4. CANCELLED FEATURES (No Longer Gaps)

The following were gaps under Epic 253 but are **cancelled** under Epic 341:

### ~~4.1 Irrigation Notifications~~ — CANCELLED
- ~~Notification backend (scheduler, APIs, suggested value, min/max range)~~
- ~~Daily irrigation logs table~~
- ~~Notification API integration (FE ↔ BE)~~
- **GitLab issues to close:** #348, #349, #350, #351, #352, #363, #364, #437, #1235

### ~~4.2 Mobile APIs~~ — CANCELLED
- ~~Push notification service~~
- ~~Mobile notification list endpoint~~
- ~~Mobile water record entry endpoint~~
- **GitLab issues to close:** #370, #371, #372, #436

### ~~4.3 Calendar 4-Color System~~ — CANCELLED
- ~~Calendar card logic with 4 color statuses~~
- ~~Notification status per day/outflow~~
- **GitLab issues to close:** #439, #1230

### ~~4.4 Third-Party Vendor Integrations~~ — DEFERRED to V2 Q2 2026
- ~~Shayp/Masgrau vendor-specific integrations~~
- ~~ETL pipelines, OAuth/webhook integrations~~
- ~~Vendor onboarding configuration UI~~

---

## 5. ARCHITECTURAL CONCERNS

### 5.1 Route Registration — RESOLVED
Water routes confirmed registered via `RouteServiceProvider.boot()` under `api/v2`.

### 5.2 Frontend Calling Non-Existent Endpoints — CLEANUP NEEDED
The Pinia store (`web/src/store/water.js`) defines API calls to endpoints that don't exist and are now **cancelled**:
- `GET /water/irrigation-notifications` → **disable/remove**
- `PATCH /water/irrigation-notifications/{id}` → **disable/remove**
- `POST /water/irrigation-notifications/batch-update` → **disable/remove**
- `PUT /water/calendar/sources/{sourceId}/status` → **evaluate** — may still be needed for simple green/red toggle

**Action:** Remove or disable these store methods to prevent silent errors in production.

### 5.3 Lynx Data Model — DECIDED
Own `lynx_*` tables — different granularity (daily/zone vs hourly/device). Lynx records feed into `WaterReading` with `is_connected_device_record = true` for dashboard display.

---

## 6. GitLab ISSUE STATUS SUMMARY (Updated for Epic 341)

### Disposition

| Action | Count | Issues |
|--------|-------|--------|
| **Fix (bugs)** | 4 | #2254, #2258, #2263, #2283 |
| **Close (cancelled bugs)** | 2 | #2259, #2268 |
| **Close (cancelled features)** | 15 | #348, #349, #350, #351, #352, #363, #364, #370, #371, #372, #436, #437, #439, #1230, #1235 |
| **Verify & promote** | 8 | #376, #377, #378, #379, #380, #381, #382, #383 |
| **Verify & merge** | 3 | #384, #438, #440 |
| **Keep for QA** | 3 | #1192, #1193, #1224 |
| **No longer applicable** | 2 | #345, #346 |
| **New (Lynx)** | 12 | Stories 3.A1–A5, 3.B1–B5, 3.C1–C2 |

### Items on staging — need production verification
- #376: Water Records API with Advanced Filters
- #377: Monthly Irrigation Data API
- #378: Update Notification Status API *(may need adjustment — notification concept changed)*
- #379: Annual Outflow Data API
- #380: ET Data API
- #381: Rainfall Data API
- #382: Database Structure for IoT Readings
- #383: Daily Totals Calculation Script

---

## 7. PRIORITY ACTION PLAN (Updated for Epic 341)

### P0 — Immediate (Blocking production)

1. **Fix 4 bugs** (#2254, #2258, #2263, #2283) — users are hitting these
2. **Remove notification UI** — dead code calling non-existent endpoints, will cause errors
3. **Simplify calendar to 2 colors** — remove light blue "planned" dots

### P1 — High (Adare Manor commitment, target end of March 2026)

4. **Build Lynx Cloud Ingest API** (stories A1–A5) — Laravel endpoints, auth, health monitoring MVP
5. **Build Lynx Sync Agent** (stories B1–B5) — Python agent, reconciliation, packaging
6. **Build minimal Back Office UI** — club config, API key management
7. **Wire Lynx data into Water Page** — zones as outflows in all views

### P2 — Medium (Feature completion for release)

8. **Verify connected meters work** — Shayp/Masgrau data, IoT vs manual, daily aggregation
9. **Promote 8 staging items to production** (#376–#383)
10. **Close cancelled GitLab issues** — 15 notification/mobile issues + 2 cancelled bugs
11. **Fix graph-data endpoint** — create or redirect

### P3 — Post-release

12. **Full health monitoring dashboard** (V2)
13. **MSI installer for agent** (April rollout ~10 clubs)
14. **Vendor onboarding architecture** (Q2 2026)
15. **Data source conflict handling** (Shayp + Lynx on same site, Q2 2026)

---

## 8. RISK REGISTER (Updated)

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Lynx pilot deadline (end March) with zero code | HIGH | HIGH | Start A1 immediately, parallelize cloud + agent workstreams |
| Notification UI left in place → user errors on non-existent endpoints | HIGH | HIGH | Remove notification UI in Phase 1, disable store methods |
| 4 open bugs degrading user experience | MEDIUM | CONFIRMED | Fix before Lynx work |
| No developer handover context | MEDIUM | CONFIRMED | This document + codebase analysis serves as handover |
| No tests found for water features | MEDIUM | HIGH | Add test coverage as part of bug fixes and Lynx work |
| Connected meter vendor strategy undefined | LOW | HIGH | Defer until after Lynx pilot succeeds |
| Agent MSI installer not ready for April rollout | MEDIUM | MEDIUM | Manual install for pilot, plan MSI for April |

---

*Report updated 2026-03-10 to align with Epic 341 (replacing Epic 253). Original gap analysis dated 2026-03-02.*
