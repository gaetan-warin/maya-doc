# Water Epic — Comprehensive Gap Analysis Report

<<<<<<< HEAD
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
=======
**Date:** 2026-03-06  
**Scope:** `core-2.0` (Laravel backend) + `web` (Vue 3 frontend)  
**Method:** direct code inspection cross-checked with the documents in `doc/waterEpic/`
>>>>>>> 8e52b688f4ba6c2d5c5ee7031c4687ec2ecb86f9

---

## Executive Summary

<<<<<<< HEAD
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
=======
The Water Epic is **partially implemented, with a stronger foundation than the previous report suggested**.

### Verified high-level state

- **Water routes are registered correctly** under `api/v2` via `RouteServiceProvider`; route registration is **not** a blocker.
- **Water source CRUD, tenant/source settings, water reading CRUD, dashboard APIs, and Water 2.0 dashboard UI are implemented.**
- **Connected water meter hourly ingestion and daily aggregation are implemented**, and Water 2.0 already has a read-only “connected meter logs” view.
- **Irrigation notifications are still frontend-only**: the Water 2.0 UI expects notification APIs that do not exist yet.
- **Calendar read APIs exist**, but **calendar status persistence does not**. The backend currently returns `irrigated`, `planned`, and `unknown` only; it cannot yet persist `confirmed` / `denied`.
- **Water-specific mobile APIs are not implemented**, but the shared push-notification foundation already exists and can be reused.
- **Toro Lynx is still zero-code in the repo**.

### Important corrections versus the previous version

1. **Route registration issue removed** — routes are already loaded from `core-2.0/routes/api/water.php`.
2. **Calendar is not “missing backend”** — the `GET /water/calendar` backend exists; what’s missing is the write path and notification-backed statuses.
3. **Mobile is not pure 0% foundation-wise** — water-specific mobile endpoints are missing, but shared Expo push infrastructure and `/api/v2/device-register` already exist.
4. **`GET /water/graph-data` is legacy debt, not a Water 2.0 blocker** — current `web/src/views/water.vue` uses the Water 2.0 dashboard APIs, not the old graph endpoint.
5. **Connected meter support is more than a raw model** — there is ingest, aggregation, hourly graph support, and read-only UI, but the “manual vs connected” distinction is still incomplete.
>>>>>>> 8e52b688f4ba6c2d5c5ee7031c4687ec2ecb86f9

---

## 1. Verified Capability Matrix

<<<<<<< HEAD
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
=======
| Capability Area | Backend | Frontend | Current Reality |
|----------------|---------|----------|-----------------|
| Water Sources & Settings | Implemented | Implemented | Usable today |
| Water Readings CRUD | Implemented | Implemented | Usable, but bug-prone |
| Dashboard & Card Modals | Mostly implemented | Implemented | Usable, with a few placeholder/partial values |
| Days Watered Calendar | Read API implemented | UI implemented | Read works; status persistence missing |
| Irrigation Notifications | Missing | Implemented | FE calls non-existent APIs |
| Connected Water Meters | Partially implemented | Partially implemented | Ingest + aggregation + hourly graphs exist; distinction/config gaps remain |
| Water-specific Mobile APIs | Missing | N/A | Reuse shared push foundation |
| Toro Lynx | Missing | Missing | No code yet |

---

## 2. What Is Implemented in Code

### 2.1 Routing & API Foundation
>>>>>>> 8e52b688f4ba6c2d5c5ee7031c4687ec2ecb86f9

**Confirmed implemented**

<<<<<<< HEAD
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
=======
- Water routes live in `core-2.0/routes/api/water.php`
- They are loaded under `api/v2` in `core-2.0/app/Providers/RouteServiceProvider.php`
- Shared device registration for push notifications exists in `core-2.0/routes/api.php`

**Conclusion:** route registration is **done**.

---

### 2.2 Water Sources Management
>>>>>>> 8e52b688f4ba6c2d5c5ee7031c4687ec2ecb86f9

**Backend**

- `WaterSource`, `WaterSourceType`, `WaterSourceDefaultAssociation`
- CRUD controllers and services
- `GET /settings/water/sources`
- `POST /settings/water/tenant/{tenant}/sources`
- `PUT /settings/water/sources/{source}`
- `DELETE /settings/water/sources/{source}`

<<<<<<< HEAD
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
=======
**Frontend**

- `web/src/components/setting/water/Index.vue`
- `web/src/components/setting/water/WaterInflow.vue`
- `web/src/components/setting/water/WaterOutflow.vue`
- Water 2.0 source selector and form usage

**Status:** **Implemented**

---

### 2.3 Tenant & Source Water Settings

**Backend**

- `TenantWaterSettings` and `SourceWaterSettings` models exist
- `GET/PUT /settings/water/tenant/{tenant}`
- `PUT /settings/water/source/{source}`
- `POST /settings/water/source/{source}/validate-period`

**Frontend**

- `web/src/components/setting/water/components/WaterConfiguration.vue`
- Reminder time and mobile notification settings are wired
- Irrigation frequency / weekdays / irrigation period support is wired in settings UI

**Status:** **Implemented**

---

### 2.4 Water Readings CRUD

**Backend**

- `WaterReadingController` supports `index`, `store`, `update`, `destroy`
- `WaterReadingService` handles meter-reading vs daily-consumption logic
- Listeners exist for site-consumption create/update/delete
- Request validation exists for list/create/update

**Frontend**

- Water 2.0 bottom section exposes manual readings CRUD
- Connected logs tab is present as read-only
- Pagination, sorting, and filters are wired in `useWaterReadings.js`

**Status:** **Implemented, but with known defects**

---

### 2.5 Water Dashboard & Card Modals

**Backend**

- `GET /water/dashboard`
- `GET /water/statistics`
- `GET /water/usage`
- `GET /water/budget`
- `GET /water/et`
- `GET /water/rainfall`
- `GET /water/calendar`

**Frontend**

- Main page: `web/src/views/water.vue`
- Cards: ET, Rainfall, Water Usage, Days Watered, Site Conditions, Water Budget
- Modals: Water Usage, ET, Rainfall, Water Budget, Days Watered

**Status:** **Mostly implemented**

**Known limitations in implemented dashboard logic**

- `WaterDashboardService` still returns `month_forecast: { value: 0, status: 'coming_soon' }` for the Water Usage card.
- The card and mapper already handle this placeholder state.

---

### 2.6 Days Watered Calendar Read Path

**Backend**

- `GET /water/calendar` exists
- `WaterUsageService::getIrrigationCalendar()` returns day-by-day data
- Current backend status generation is:
  - `irrigated` when actual consumption exists
  - `planned` when the irrigation schedule says the day is planned
  - `unknown` otherwise

**Frontend**

- `DaysWateredModalContent.vue`
- 4-color UI logic exists in the frontend composables
- Green days are protected from editing in FE logic

**Status:** **Partially implemented**

**Critical nuance**

- The backend currently **does not persist or emit** notification-driven `confirmed` / `denied` states because there is no irrigation daily-log / notification table yet.
- This means the FE supports more states than the BE can currently provide.

---

### 2.7 Connected Water Meter Foundation

**Backend**

- `ConnectedWaterMeterDevice` model exists
- `ConnectedWaterMeterHourlyRecord` model exists
- `POST /water/hourly-records` exists
- `ConnectedWaterMeterService` normalizes and upserts hourly records
- `AggregateConnectedWaterMeterDailyReadings` command exists
- `ConnectedWaterMeterDailyAggregationService` creates daily `water_readings` from hourly data
- Hourly usage graph support exists in `WaterUsageService` / `WaterSourceRepository`

**Frontend**

- Water 2.0 shows a separate read-only “Connected Meter Logs” tab
- Hourly graphs are available when connected sources exist
- Sources are tagged with `connected_water_meter_device`

**Status:** **Partially implemented**

---

### 2.8 Shared Push Notification Foundation

**Backend foundation already present**

- `PushNotificationService.php`
- `PushNotificationController.php`
- `POST /api/v2/device-register`
- `user_push_tokens` table
- Existing non-water push-processing command flow

**Status:** **Implemented as shared infrastructure, not yet used by water**

---

### 2.9 Data Migration / Infrastructure

- **21 water-related migrations** exist
- **9 commands** exist under `core-2.0/app/Console/Commands/Water`
- **1 additional water command** exists for connected-meter daily aggregation

**Status:** **Implemented**

---

## 3. What Is Partially Implemented

### 3.1 Irrigation Notifications — FE Built, BE Missing

**Current frontend state**

- `NotificationCard.vue`
- `BulkConsumptionPanel.vue`
- `useWaterNotifications.js`
- Store methods:
  - `fetchIrrigationNotifications()`
  - `updateIrrigationNotification()`
  - `batchUpdateIrrigationNotifications()`

**Expected backend endpoints do not exist**

- `GET /water/irrigation-notifications`
- `PATCH /water/irrigation-notifications/{id}`
- `POST /water/irrigation-notifications/batch-update`

**Additional nuance**

- The FE expects notification payloads enriched with:
  - `water_source`
  - `prev_reading`
  - `next_reading`
  - `suggested_value`

**Status:** **Frontend complete enough to integrate; backend not started**

---

### 3.2 Calendar Status Persistence — Write Path Missing

**What exists**

- Calendar GET API exists
- Side panel UI exists
- FE interaction rules exist

**What is missing**

- `PUT /water/calendar/sources/{sourceId}/status`
- A persistence model/table for daily irrigation decisions
- Merge logic combining:
  - actual readings
  - planned schedule
  - confirmed/denied notification states

**Frontend nuance**

- `useSidePanel.js` already calls `waterStore.updateIrrigationStatus(...)`
- `useDateClickHandler.js` still has a TODO and does not persist inline changes

**Status:** **Partially implemented**

---

### 3.3 Connected Meter “Manual vs Connected” Distinction

**What exists**

- Schema field: `water_readings.is_connected_device_record`
- Filters in repository logic
- UI separation between manual and connected log tabs

**What is still incomplete**

- `ConnectedWaterMeterDailyAggregationService` creates daily `water_readings`, but it does **not** set `is_connected_device_record = true`
- Because of that, read-only connected logs and manual/connected separation cannot be considered fully trustworthy yet
- There is no device management API/UI for creating and managing `connected_water_meter_devices`

**Status:** **Partially implemented**

---

### 3.4 Dashboard Completeness

**What works**

- All six cards render
- Core dashboard response structure exists
- ET/rainfall/site conditions/budget are populated

**Still partial**

- Water Usage “month forecast” is placeholder only (`coming_soon`)
- Notification panel is mounted by default in the top section and currently depends on missing APIs

**Status:** **Mostly implemented, not feature-complete**

---

### 3.5 Legacy Water V1 API Mismatch

There is still frontend code in the legacy `web/src/components/water/*` path that calls:

- `GET /water/graph-data`

That endpoint does **not** exist in the current backend.

**Important scope note**

- This is **not** blocking the current Water 2.0 page in `web/src/views/water.vue`
- It is legacy debt or a compatibility gap, not the main Water 2.0 blocker

**Status:** **Legacy mismatch**

---

### 3.6 Test Coverage

The repo contains automated tests in general, including push-notification tests, but:

- **no water-specific backend tests**
- **no water-specific frontend tests**

**Status:** **Missing water test coverage**

---

## 4. What Is Missing

### 4.1 Irrigation Notifications Backend

Missing pieces:

- `irrigation_daily_logs` / equivalent table
- backend model / repository / service
- notification generation scheduler
- single update API
- batch update API
- list API
- suggested value calculation
- min/max validation support for notification entry

**Status:** **Missing**

---

### 4.2 Water-Specific Mobile APIs

Missing pieces:

- Water mobile notification list endpoint
- Water mobile submit/confirm endpoint
- Water-specific push scheduling job/command
- Water mobile API contract/documentation

**Status:** **Missing**

**Note:** shared push infrastructure already exists and should be reused.

---

### 4.3 Toro Lynx Connector

Missing pieces:

- cloud tables (`lynx_*`)
- config/auth/API key management
- ingest endpoint
- sync logs / health dashboard
- agent project
- reconciliation engine
- Windows packaging

**Status:** **Missing**

---

### 4.4 Third-Party Vendor Integrations

Still missing:

- Migration of the existing Shayp integration from NiFi into the Python orchestrator
- Migration of the existing Masgrau integration from NiFi into the Python orchestrator
- vendor onboarding flow
- provider abstraction completion

**Important note**

- Shayp and Masgrau should not be treated as greenfield vendor integrations.
- They already exist in NiFi and need to be migrated into the Python orchestrator target architecture.
- The main delivery need is migration plus parity validation, not re-discovery of vendor requirements from scratch.

**Status:** **Missing in the target Python orchestrator architecture**

---

### 4.5 Water-Specific Operational Monitoring

Missing pieces:

- water sync health monitoring
- connected-meter anomaly checks
- notification scheduler monitoring
- Lynx health monitoring / alerts

**Status:** **Missing**

---

## 5. Updated Risk View

| Risk | Impact | Likelihood | Updated Note |
|------|--------|------------|--------------|
| Notification UI calls missing APIs | HIGH | HIGH | Immediate user-facing gap in Water 2.0 |
| Calendar write path missing | HIGH | HIGH | UI supports actions that backend cannot persist |
| Connected-meter readings not clearly flagged | MEDIUM | HIGH | Can blur manual vs connected reporting |
| Water-specific tests absent | MEDIUM | HIGH | Bug-fix work has no safety net |
| NiFi vendor migration underestimated | MEDIUM | HIGH | Shayp and Masgrau already exist outside the target architecture and need migration parity, cutover, and rollback planning |
| Lynx pilot with zero code | HIGH | HIGH | Still a greenfield module |
| Legacy `graph-data` mismatch | LOW | MEDIUM | Only matters if legacy water v1 stays in scope |

---

## 6. Recommended Delivery Order

1. **Stabilize existing water reading bugs**
2. **Implement notification data model + APIs**
3. **Implement calendar persistence on top of the notification/daily-log model**
4. **Fix connected-meter generated-reading flagging**
5. **Add water-specific mobile APIs using the existing push foundation**
6. **Migrate Shayp and Masgrau from NiFi into the Python orchestrator**
7. **Build Lynx only after the Water 2.0 core workflow is stable**

---

## 7. Bottom Line

The repo already contains a **working Water 2.0 base**:

- routes
- CRUD
- dashboard
- settings
- connected-meter hourly ingestion
- shared push infrastructure

The real missing work is **not** the foundation. The missing work is:

- **notifications**
- **calendar write/persistence**
- **final connected-meter distinction**
- **mobile water APIs**
- **Shayp / Masgrau migration from NiFi to Python orchestrator**
- **Lynx**

That should now drive the implementation plan.
>>>>>>> 8e52b688f4ba6c2d5c5ee7031c4687ec2ecb86f9
