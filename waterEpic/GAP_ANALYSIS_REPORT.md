# Water Epic — Comprehensive Gap Analysis Report

**Date:** 2026-03-06  
**Scope:** `core-2.0` (Laravel backend) + `web` (Vue 3 frontend)  
**Method:** direct code inspection cross-checked with the documents in `doc/waterEpic/`

---

## Executive Summary

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

---

## 1. Verified Capability Matrix

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

**Confirmed implemented**

- Water routes live in `core-2.0/routes/api/water.php`
- They are loaded under `api/v2` in `core-2.0/app/Providers/RouteServiceProvider.php`
- Shared device registration for push notifications exists in `core-2.0/routes/api.php`

**Conclusion:** route registration is **done**.

---

### 2.2 Water Sources Management

**Backend**

- `WaterSource`, `WaterSourceType`, `WaterSourceDefaultAssociation`
- CRUD controllers and services
- `GET /settings/water/sources`
- `POST /settings/water/tenant/{tenant}/sources`
- `PUT /settings/water/sources/{source}`
- `DELETE /settings/water/sources/{source}`

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
