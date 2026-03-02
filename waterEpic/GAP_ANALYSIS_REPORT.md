# Water Epic — Comprehensive Gap Analysis Report

**Date:** 2026-03-02
**Scope:** Core 2.0 (Laravel backend) + Web (Vue 3 frontend)
**Source documents:** Status_epic.md, technical-documentation.md, HANDOVER-LYNX-CONNECTOR.md, MSM call transcript, Epic 253 (GitLab HTML export — 50 issues)

---

## Executive Summary

The Water Epic is **partially implemented**. The core water management foundation (sources, readings, consumption tracking, dashboard) is solid, but several critical modules are incomplete or entirely missing. The documentation describes **4 major capability areas**, and the current state is:

| Capability Area | Backend | Frontend | Overall |
|----------------|---------|----------|---------|
| Water Sources & Readings (CRUD) | ~90% | ~85% | ~87% |
| Dashboard & Analytics (Cards, Charts) | ~80% | ~80% | ~80% |
| Irrigation Notifications & Calendar | ~30% | ~60% | ~45% |
| Connected Meters & Lynx Connector | ~15% | ~0% | ~8% |
| Mobile APIs | 0% | N/A | 0% |

**Critical finding:** Water API routes defined in `core-2.0/routes/api/water.php` may not be registered in the main router — needs verification.

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
| Top section (6 cards + notifications) | DONE | `web/src/components/water2/sections/WaterTopSection.vue` |
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
| ET Factor, notifications toggle, reminder time | DONE | In WaterConfiguration.vue |

### 1.5 Connected Water Meter (Basic Model Only)
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

## 2. WHAT IS PARTIALLY IMPLEMENTED (Gaps Within Existing Features)

### 2.1 Irrigation Notifications — Frontend Built, Backend Missing

**Severity: HIGH**

The frontend has a complete notification UI but the backend APIs don't exist.

| Component | FE Status | BE Status | Gap |
|-----------|-----------|-----------|-----|
| Notification list UI (past 7 days) | DONE | MISSING | No `GET /water/irrigation-notifications` endpoint |
| Multi-select & select all controls | DONE | MISSING | No backend to call |
| Bulk mark Yes/No (no value) | DONE | MISSING | No `POST /water/irrigation-notifications/batch-update` |
| Bulk mark Yes/No with values | DONE | MISSING | No bulk value distribution endpoint |
| Single notification expansion + input | DONE | MISSING | No `PATCH /water/irrigation-notifications/{id}` |
| NotificationCard.vue | DONE | — | UI complete but calling non-existent endpoints |
| BulkConsumptionPanel.vue | DONE | — | Panel built but no API integration |

**GitLab Issues (all TODO):**
- #348: Create database table for Daily Irrigation Logs
- #349: Create API: Fetch Notifications (Past 7 Days, Per Outflow)
- #350: Create API: Bulk Mark Yes/No (No Value)
- #351: Create API: Bulk Mark Yes/No With Values (Distribution)
- #352: Create API: Submit Single Notification Response with Value
- #363: Implement Suggested Value Calculation Service/API
- #364: Implement Min/Max Range Service/API for Expanded Notification
- #437: Update Notification Scheduler/Generator per Outflow Settings

**What's needed:**
1. Create `irrigation_notifications` (or `daily_irrigation_logs`) database table + migration
2. Create IrrigationNotification model
3. Create notification generation scheduler (daily cron based on outflow irrigation settings)
4. Create API controller with endpoints: list, single update, batch update
5. Implement suggested value calculation service
6. Implement min/max range validation service
7. Wire up existing frontend to real API

### 2.2 Days Watered Calendar — Status Changes Not Persisted

**Severity: HIGH**

| Component | Status | Details |
|-----------|--------|---------|
| Calendar UI (month view, color codes) | DONE | `DaysWateredModalContent.vue` |
| Outflow filter dropdown | DONE | Working |
| Status color legend | DONE | green/red/blue/light-blue |
| Side panel for status selection | DONE | `OutflowStatusPanel.vue` |
| **API call to save status change** | **TODO** | `useDateClickHandler.js:46` has `// TODO: Call API to save the status change` |
| Calendar status update API | PARTIAL | `PUT /water/calendar/sources/{sourceId}/status` referenced in store but not confirmed in backend |
| Calendar card logic (4 color statuses) | TODO | GitLab #1230: still TODO |

**GitLab Issues:**
- #439: Extend Days Watered APIs: Return Notification Status Per Day/Outflow (TODO)
- #440: Create Days Watered API: Update Notification Status from Calendar (developer testing)
- #1230: Implement Calendar Card Logic with four colour code statuses (TODO)

### 2.3 Notification API Integration (Frontend ↔ Backend)

**Severity: HIGH**

| Component | Status | Details |
|-----------|--------|---------|
| FE store methods defined | DONE | `fetchIrrigationNotifications()`, `updateIrrigationNotification()`, `batchUpdateIrrigationNotifications()` in `water.js` |
| API integration wiring | BLOCKED | GitLab #1235: "Implement API Integration for Notifications (FE)" — status: **BLOCKED** |
| Validation rules (FE) | PARTIAL | Basic status validation exists, comprehensive rules missing |

### 2.4 Water Record Bugs (Known Issues)

| Bug # | Description | Status |
|-------|-------------|--------|
| #2254 | Water Data Management table — cannot delete records | Bug |
| #2258 | Water Data Management table — cannot update records | Bug |
| #2259 | Days Watered Calendar — inappropriate error on status change | Bug |
| #2263 | Water Meter reading add/edit — previous reading value wrong | Bug |
| #2268 | Irrigation Notifications are not generating | Bug |
| #2283 | Irrigation calendar — total consumption value wrongly updated | Bug |

### 2.5 Connected Meter — APIs Need Update for IoT vs Manual Distinction

| Component | Status | Details |
|-----------|--------|---------|
| Extend APIs to identify IoT vs Manual | IN REVIEW | GitLab #384: "developer in review" |
| Water Records Ingestion for Distributed Bulk Values | TESTING | GitLab #438: "developer testing" |

### 2.6 Unified Table Structure for Connected Meters

| Component | Status | Details |
|-----------|--------|---------|
| Design unified table structure | IN PROGRESS | GitLab #427 |
| Implement unified table in DB | TODO | GitLab #428 |
| Migrate existing data to unified table | TODO | GitLab #429 |
| Update APIs for unified table | TODO | GitLab #430 |
| Update cron jobs for daily totals | TODO | GitLab #431 |

---

## 3. WHAT IS COMPLETELY MISSING

### 3.1 Toro Lynx Connector — Entire Module

**Severity: CRITICAL (committed to Adare Manor, target end of March 2026)**

**Zero code exists in the codebase.** The handover document describes a full two-sided architecture:

#### 3.1a Cloud Ingest API (Laravel side) — NOT STARTED
| Story | Description | Status |
|-------|-------------|--------|
| A1 | API contract & data model (JSON schemas, DB migrations) | NOT STARTED |
| A2 | Club config & API key management (Back Office UI + middleware) | NOT STARTED |
| A3 | Ingest API endpoint (`POST /api/v2/lynx/sync`) | NOT STARTED |
| A4 | Health monitoring (dashboard, alerts, sync logs) | NOT STARTED |
| A5 | Data access layer for Water Page (feeding Epic 253) | NOT STARTED |

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
| B5 | Agent: Windows installation package (.msi) | NOT STARTED |

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

#### 3.1c Back Office UI (Vue) — NOT STARTED
- Lynx Connectors list (all clubs, sync status)
- Club config form (tenant, units, timezone, irrigation day start, API key generation)
- Sync log viewer (per-club, last 30 syncs)
- Health alerts display

### 3.2 Mobile APIs — Entire Module

**Severity: HIGH**

| Story | Description | Status |
|-------|-------------|--------|
| #370 | Push Notification Generation API (Expo-based) | TODO |
| #371 | Create New Mobile Notification Service | TODO |
| #372 | Extend "Add Water Records" API for Mobile entries | TODO |
| #436 | Create Mobile Notifications List Endpoint | TODO |

**Required components (none exist):**
- Mobile push notification service (Expo integration)
- Mobile-specific notification list endpoint
- Mobile water record entry endpoint
- Push notification scheduling based on `preferred_reminder_time`

### 3.3 Third-Party Water Meter Integrations (Shayp, Masgrau, etc.)

**Severity: MEDIUM**

The connected meter model exists but is **completely generic** — no vendor-specific integrations:
- Zero references to "Shayp" in codebase
- Zero references to "Masgrau" in codebase
- No ETL pipelines for external data ingestion
- No OAuth/webhook integrations with vendors
- No vendor onboarding configuration UI
- Strategic direction still undefined (per Status_epic.md)

### 3.4 Health Monitoring & Alerting System

**Severity: MEDIUM**

No water-specific health monitoring exists:
- No sync health tracking
- No alert on missed data syncs
- No data quality checks
- No anomaly detection
- No operational dashboard for water system health

---

## 4. ARCHITECTURAL CONCERNS

### 4.1 Route Registration Issue
The water routes in `core-2.0/routes/api/water.php` may not be properly included in the main `routes/api.php`. This needs **immediate verification** — if routes aren't registered, the entire water API is unreachable.

### 4.2 Frontend Calling Non-Existent Endpoints
The Pinia store (`web/src/store/water.js`) defines API calls to endpoints that don't exist in the backend:
- `GET /water/irrigation-notifications`
- `PATCH /water/irrigation-notifications/{id}`
- `POST /water/irrigation-notifications/batch-update`
- `PUT /water/calendar/sources/{sourceId}/status`

This means the notification UI will fail silently or show errors when users interact with it.

### 4.3 Key Decision Pending: Lynx Data Model
The handover document (Section 5, Q1) asks: **Can Lynx data use Epic 253's connected meter tables?**
- If YES → Lynx becomes another source in existing pipeline (simpler)
- If NO → Dedicated `lynx_water_records` tables needed

This decision hasn't been made and blocks the entire Lynx Connector implementation.

---

## 5. GitLab ISSUE STATUS SUMMARY

### By Status (50 total issues):

| Status | Count | Issues |
|--------|-------|--------|
| Deployed on stage | 8 | #376, #377, #378, #379, #380, #381, #382, #383 |
| Developer testing | 8 | #438, #440, #1040, #1041, #1042, #1058, #1059, #1229, #1233, #1234 |
| Developer in review | 1 | #384 |
| In progress | 1 | #427 |
| TODO (not started) | 14 | #348, #349, #350, #351, #352, #363, #370, #371, #372, #428, #429, #430, #431, #436, #437, #439, #1224, #1230 |
| Blocked | 2 | #346, #1235 |
| Bug | 6 | #2254, #2258, #2259, #2263, #2268, #2283 |
| Unspecified | ~10 | #345, #364, #1192, #1193, etc. |

### Items "deployed on stage" that need production verification:
- #376: Water Records API with Advanced Filters
- #377: Monthly Irrigation Data API
- #378: Update Notification Status API
- #379: Annual Outflow Data API
- #380: ET Data API
- #381: Rainfall Data API
- #382: Database Structure for IoT Readings
- #383: Daily Totals Calculation Script

---

## 6. PRIORITY ACTION PLAN

### P0 — Immediate (Blocking production and commitments)

1. **Verify water route registration** in `routes/api.php` — if broken, nothing works
2. **Fix 6 known bugs** (#2254, #2258, #2259, #2263, #2268, #2283) — users are hitting these
3. **Complete irrigation notification backend** (#348-#352, #363, #364, #437) — FE is built and waiting

### P1 — High (Adare Manor commitment, target March/April 2026)

4. **Make Lynx data model decision** (Q1 from handover doc) — blocks everything below
5. **Build Lynx Cloud Ingest API** (stories A1-A5) — Laravel endpoints, auth, health monitoring
6. **Build Lynx Sync Agent** (stories B1-B5) — Python agent, reconciliation, packaging
7. **Build Lynx Back Office UI** — club config, sync logs, health dashboard

### P2 — Medium (Feature completion)

8. **Complete calendar status persistence** (fix TODO in `useDateClickHandler.js`)
9. **Finish unified connected meter table structure** (#427-#431)
10. **Complete IoT vs Manual reading distinction** (#384, #438)
11. **Promote staging items to production** (#376-#383)

### P3 — Lower (Mobile & future integrations)

12. **Design and build mobile APIs** (#370-#372, #436)
13. **Plan third-party meter vendor integrations** (Shayp, Masgrau)
14. **Build health monitoring system**

---

## 7. RISK REGISTER

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Lynx pilot deadline (end March) with zero code | HIGH | HIGH | Start A1 immediately, parallelize cloud + agent |
| Water routes not registered = entire API broken | CRITICAL | MEDIUM | Verify immediately |
| FE calling non-existent BE endpoints = user errors | HIGH | HIGH | Implement notification backend or add graceful fallbacks |
| No developer handover context | MEDIUM | CONFIRMED | This document + codebase analysis serves as handover |
| 6 open bugs degrading user experience | MEDIUM | CONFIRMED | Fix before adding new features |
| No tests found for water features | MEDIUM | HIGH | Add test coverage as part of bug fixes |
| Connected meter vendor strategy undefined | LOW | HIGH | Defer until after Lynx pilot succeeds |

---

*Report generated from full codebase analysis of `core-2.0/` and `web/` directories cross-referenced against all documentation in `waterEpic/`.*
