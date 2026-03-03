# Water Epic — Implementation Plan

**Date:** 2026-03-02 | **~50–53 dev days / ~11 weeks** (1 dev) | **~9 weeks** (2 devs)
**Ref:** `GAP_ANALYSIS_REPORT.md` | **Critical path:** Phase 0 → 1 → 4 → 5

---

## Checklist

### Decisions (before starting)
- [x] **D1** Lynx data model → **own `lynx_*` tables (decided)** — different granularity (daily/zone vs hourly/device), needs auto/manual volume split + sync audit trail. Both feed into `water_readings` for dashboard.
- [x] **D2** Lynx agent repo → **separate `maya-lynx-agent` (decided)** — deployed as a self-contained installer (`.exe` via PyInstaller or `.msi`) on customer's Windows machine at the golf club. Zero shared code with Laravel/Vue. Simple install: drop to `C:\MayaLynx\`, configure `config.yaml`, register in Windows Task Scheduler.
- [x] **D3** Adare Manor backfill depth → **configurable per club (decided)** — agent `--backfill --months N` flag pulls N months of history from Lynx `water_use` table, tagged as `data_source = 'backfill'`. Default 12 months. Configurable in `config.yaml` per club. Historical data uses midnight boundaries (not irrigation day) — acceptable trade-off for past data.
- [x] **D4** Mobile push provider → **Expo Push (confirmed)** — already in production for tasks/incidents/spraying via `PushNotificationService.php`. Device tokens stored in `user_push_tokens`, registration via `POST /api/v2/device-register`.
- [x] **D5** Notification table name → **`irrigation_daily_logs` (decided)** — matches GitLab #348, reflects permanent daily log per outflow (not a transient notification).

### Phase 0 — Stabilization *(3–4 days, Week 1)*
- [ ] **0.1** Fix bug #2254 — Cannot delete water records
- [ ] **0.2** Fix bug #2258 — Cannot update water records
- [ ] **0.3** Fix bug #2263 — Previous reading value wrong in message
- [ ] **0.4** Fix bug #2283 — Calendar total consumption wrongly updated
- [ ] **0.5** Close #2259 + #2268 as "needs Phase 1" (not bugs, missing features)

### Phase 1 — Irrigation Notifications & Calendar *(12–14 days, Weeks 2–4)*
- [ ] **1.1** Create irrigation notifications migration (0.5d)
- [ ] **1.2** Create model + repository + service provider binding (0.5d)
- [ ] **1.3** Build notification generation scheduler command (2d)
- [ ] **1.4** Build notification APIs — list/update/batch (4–5d)
- [ ] **1.5** Build calendar status save endpoint (1d)
- [ ] **1.6** Wire frontend to real APIs + test (3d)
- [ ] **1.7** Fix missing `GET /water/graph-data` endpoint (0.5–1d)

### Phase 2 — Mobile API Layer *(5.5 days, Weeks 3–5, overlaps Phase 1)*
- [ ] **2.1** Build mobile push notification service + scheduler (3d)
- [ ] **2.2** Build mobile notification list endpoint (1d)
- [ ] **2.3** Build mobile water record entry endpoint (1d)
- [ ] **2.4** Write API documentation for mobile team (0.5d)

### Phase 3 — Connected Meter Improvements *(6 days, Weeks 5–6)*
- [ ] **3.1** Finalize unified table design (1d)
- [ ] **3.2** Create unified table migration (1d)
- [ ] **3.3** Migrate existing data (1d)
- [ ] **3.4** Update APIs for unified table (1.5d)
- [ ] **3.5** Update daily aggregation cron (0.5d)
- [ ] **3.6** Complete IoT vs manual distinction in UI (1d)

### Phase 4 — Toro Lynx Connector *(16.5 days, Weeks 6–10)*
- [ ] **4.A1** Define API contract + create DB migrations (1d)
- [ ] **4.A2** Build club config + API key management (1.5d)
- [ ] **4.A3** Build ingest API endpoint `POST /api/v2/lynx/sync` (1.5d)
- [ ] **4.A4** Build health monitoring + Slack alerts (1.5d)
- [ ] **4.A5** Wire Lynx data into Water Page dashboard (1.5d)
- [ ] **4.B1** Python: Lynx DB query layer via pyodbc (1.5d) *(parallel with A)*
- [ ] **4.B2** Python: Reconciliation engine (2d)
- [ ] **4.B3** Python: HTTPS push to Maya API (1d)
- [ ] **4.B4** Python: CLI + scheduling + logging (1d)
- [ ] **4.B5** Python: PyInstaller Windows package (1d)
- [ ] **4.C1** End-to-end integration test (1.5d)
- [ ] **4.C2** Adare Manor pilot deployment (1.5d)

### Phase 5 — QA & Hardening *(5 days, Weeks 10–11)*
- [ ] **5.1** Promote 8 staging items to production (2d)
- [ ] **5.2** Execute GitLab test cases (2d)
- [ ] **5.3** Full regression testing (1d)

### Phase 6 — Vendor Strategy *(2 days, Week 11+, planning only)*
- [ ] **6.1** Document vendor onboarding architecture
- [ ] **6.2** Prioritize Shayp / Masgrau / others by customer demand

---

## Context

- **Developers left** — no verbal handover, only codebase + documents
- **Routes confirmed** registered via `RouteServiceProvider.boot()` under `api/v2`
- **Mobile = external dependency** — API work must start early to unblock mobile team
- **Lynx Connector** — Adare Manor pilot, deadline end of April 2026
- **Lynx data model** — should NOT reuse connected meter tables (different granularity: daily zone-level vs hourly device-level)

---

## Phase 0 — Stabilization

**Goal:** Fix what's broken for current users. No new features.

| Bug | Issue | Investigation path |
|-----|-------|--------------------|
| #2254 | Can't delete water records | `WaterReadingController@destroy` exists → check `WaterReadingService.deleteWaterReading()` + `DeleteWaterSiteConsumption` listener cascade + FE store error handling |
| #2258 | Can't update water records | `WaterReadingController@update` exists → check `WaterReadingService.updateWaterReading()` validation (meter reading can't go below previous) |
| #2263 | Wrong previous reading value | Check `WaterReadingService.calculateWaterReadingValues()` or FE form "previous reading" hint |
| #2283 | Calendar total consumption wrong | Check `WaterUsageService.getIrrigationCalendar()` + `dashboardDataMapper.mapDaysWateredData()` in FE |
| #2259 | Calendar error on status change | Not a bug — `PUT /water/calendar/sources/{sourceId}/status` doesn't exist yet → Phase 1.5 |
| #2268 | Notifications not generating | Not a bug — no notification system exists at all → Phase 1.3 |

---

## Phase 1 — Irrigation Notifications & Calendar

**Goal:** Complete notification system end-to-end. FE is built and waiting for BE.

### 1.1 — Notification table migration

```
irrigation_daily_logs:
  id, tenant_id (binary UUID), water_source_id (FK, outflow only),
  notification_date, status (pending|confirmed|denied),
  consumption_value (decimal nullable), suggested_value (decimal nullable),
  source (schedule|manual), responded_at, responded_by (FK users),
  notes (text nullable), version (default 0), timestamps
  UNIQUE: (tenant_id, water_source_id, notification_date)
```

### 1.2 — Model + Repository

`IrrigationNotification` model → `belongsTo(WaterSource)`, `belongsTo(Tenant)`. Repository interface + implementation, register in service provider.

### 1.3 — Generation scheduler

Artisan command `GenerateIrrigationNotifications` (daily cron):
- Per tenant with `mobile_notifications_enabled = true` → per active outflow with `SourceWaterSettings` → check if date is in irrigation period + matches frequency (`daily`, `every_other_day`, `specific_weekdays`, `once_per_week`) → create `pending` notification
- Calculate `suggested_value`: annual allowance ÷ remaining irrigation days, adjusted by ET correction factor + recent consumption patterns
- **Past-date notifications:** define and implement behavior when the scheduler or a manual action targets a date in the past (e.g. auto-confirm, skip, or create as "late pending"). Document the chosen rule before coding.
- **Legacy data filtering:** exclude legacy/pre-migration records from notification generation — define criteria (e.g. records before a cutoff date, records without a valid `SourceWaterSettings`, or tenants not yet onboarded to Water 2.0) and add filtering to the query so no spurious notifications are created.
- **Resolves #2268**

### 1.4 — Notification APIs

| Method | Route | FE store method | GitLab |
|--------|-------|-----------------|--------|
| GET | `/water/irrigation-notifications` | `fetchIrrigationNotifications()` | #348–#352 |
| PATCH | `/water/irrigation-notifications/{id}` | `updateIrrigationNotification()` | #363, #364 |
| POST | `/water/irrigation-notifications/batch-update` | `batchUpdateIrrigationNotifications()` | #437 |

Service methods: `getNotificationsForTenant()`, `updateNotification()`, `batchUpdate()`, `calculateSuggestedValue()`, `calculateMinMaxRange()`. Plus 3 request validation classes.

### 1.5 — Calendar status save

`PUT /water/calendar/sources/{sourceId}/status` — body: `{ date, status: "confirmed"|"denied"|null }` — creates/updates `IrrigationNotification` for source+date. **Resolves #2259** and unblocks `useDateClickHandler.js:46` TODO.

### 1.6 — Frontend wiring

1. Wire API call in `useDateClickHandler.js:46`
2. Test `NotificationCard.vue` against real API
3. Test `BulkConsumptionPanel.vue` value distribution
4. Implement calendar 4-color logic (#1230)
5. Complete FE validation rules

**Resolves:** #1230, #1235, #1040–#1042, #1058, #1059

### 1.7 — Graph-data endpoint

FE store calls `GET /water/graph-data` (water.js:118) → either create in `WaterController` or redirect FE to existing `/water/usage`.

---

## Phase 2 — Mobile API Layer

**Goal:** Unblock mobile team. Start after Step 1.4 (reuses `IrrigationNotification` model/service).

### 2.1 — Push notification service

No new push provider needed — reuse existing Expo Push infrastructure.

**Existing infrastructure (already in production):**
- `PushNotificationService.php` → sends to `https://exp.host/--/api/v2/push/send`
- `user_push_tokens` table → stores `ExponentPushToken[...]` per user/device
- `POST /api/v2/device-register` → mobile app already registers tokens on startup
- Deduplication (30s cache + in-request), 7-language localization, error logging all built-in

**New work:**
- `SendIrrigationPushNotifications` Artisan command (runs every minute or via scheduler)
- Query pending `irrigation_daily_logs` where tenant's `preferred_reminder_time` has arrived
- Call existing `PushNotificationService::sendPushNotification()` with payload:
  ```json
  { "title": "Irrigation Reminder", "body": "You have N pending irrigation logs for today",
    "data": { "task_type": "irrigation", "notification_date": "YYYY-MM-DD" } }
  ```
- Respects `mobile_notifications_enabled` + `user.enable_push_notification`

**GitLab:** #370, #371

### 2.2 — Mobile notification list

`GET /mobile/water/notifications` — pending notifications for authenticated tenant. Reuses `IrrigationNotificationService`. **GitLab:** #436

### 2.3 — Mobile water record entry

`POST /mobile/water/readings` — accepts optional `notification_id`, auto-confirms notification on submit. **GitLab:** #372

### 2.4 — API documentation

Endpoints, request/response examples, auth, push payload format, error codes. Hand to mobile team immediately.

---

## Phase 3 — Connected Meter Improvements

**Goal:** Unified table structure + IoT vs manual distinction.

| Step | Work | GitLab | Effort |
|------|------|--------|--------|
| 3.1 | Finalize pluggable data source design (add `data_source`/`provider` field, keep existing tables) | #427 | 1d |
| 3.2 | Create migration for unified structure | #428 | 1d |
| 3.3 | Write + test data migration command | #429 | 1d |
| 3.4 | Update controllers/services/repos, ensure FE backward compat | #430 | 1.5d |
| 3.5 | Update `AggregateConnectedWaterMeterDailyReadings` for new structure | #431 | 0.5d |
| 3.6 | Finalize IoT vs manual distinction in Water Page (#384 in review, #438 in testing) | #384, #438 | 1d |

---

## Phase 4 — Toro Lynx Connector

**Goal:** Full Lynx integration for Adare Manor pilot. Two parallel workstreams.

### Workstream A — Cloud Ingest API (Laravel)

**4.A1 — API Contract + Data Model (1d)**

Sync payload schema:
```json
{
  "sync_id": "uuid", "club_identifier": "adare-manor", "agent_version": "1.0.0",
  "sync_date": "2026-03-15", "unit": "m3",
  "records": [{
    "zone_code": "1GR", "zone_name": "Hole 1 Green", "irrigation_date": "2026-03-14",
    "total_volume": 45.2, "auto_volume": 42.1, "manual_volume": 3.1,
    "data_source": "actual", "station_count": 8
  }]
}
```

Three tables:
- **`lynx_club_configs`** — tenant_id, club_identifier (unique), api_key_hash, unit, timezone, irrigation_day_start, last_sync_at/status, timestamps
- **`lynx_water_records`** — lynx_club_config_id (FK), zone_code, zone_name, irrigation_date, volumes (total/auto/manual), data_source (actual|scheduled|backfill), sync_id, water_reading_id (FK nullable). UNIQUE: (config_id, zone_code, irrigation_date)
- **`lynx_sync_logs`** — lynx_club_config_id (FK), sync_id, agent_version, records received/accepted/rejected, status, error_details (json), duration_ms

**4.A2 — Club Config + API Key Management (1.5d)**

`LynxClubConfig` model. Back Office Vue page: list/create/edit clubs, generate API key (shown once, stored bcrypt). `X-Lynx-Api-Key` auth middleware.

**4.A3 — Ingest Endpoint (1.5d)**

`POST /api/v2/lynx/sync` — authenticate via API key → validate payload → upsert `lynx_water_records` → create/update `WaterReading` (with `is_connected_device_record = true`) → log to `lynx_sync_logs` → return accept/reject counts.

**4.A4 — Health Monitoring (1.5d)**

Daily scheduled job: flag clubs with no sync in >26h → Slack alert. Back Office page: club health dashboard (green/yellow/red) + sync log viewer (last 30 syncs).

**4.A5 — Water Page Integration (1.5d)**

Lynx records → `WaterReading` tagged as connected device. Each zone maps to a `WaterSource` (outflow), auto-created on first sync or configured in club config. Verify dashboard/charts/calendar/budget display.

### Workstream B — Sync Agent (Python, parallel with A)

**4.B1 — DB Query Layer (1.5d):** Project setup (`maya-lynx-agent`), SQL Server via pyodbc, query `water_use_upload` (actuals, 7-day window) + `schedule_activity_download` (scheduled fallback), join `station` on SUID, config from YAML.

**4.B2 — Reconciliation Engine (2d):** Per zone per irrigation day: actuals → scheduled → skip. Station→zone aggregation (parse `1GR2` → `1GR`). Irrigation day boundary (3:55 PM configurable). Volume = `(duration/60) × station_flow`. Manual = total - auto. Edge cases: overseed, zero-duration rain hold, re-runs (upsert).

**4.B3 — HTTPS Push (1d):** Build JSON matching A1 schema, POST with retry (3×, exponential backoff), handle responses + errors.

**4.B4 — CLI + Scheduling (1d):** Flags: `--config`, `--dry-run`, `--backfill`, `--verbose`. Exit codes: 0=success, 1=partial, 2=API fail, 3=DB fail, 4=config error. Rotating log (10MB×5). Windows Task Scheduler docs.

**4.B5 — Windows Package (1d):** PyInstaller → single `.exe`. Install to `C:\MayaLynx\`. Template `config.yaml`. Installation docs for MSM tech / club IT.

### Integration

**4.C1 — E2E Test (1.5d):** Mock Lynx DB → Agent → API → Water Page. Test: actuals only, scheduled fallback, mixed, gaps, overseed, rain hold, re-sync.

**4.C2 — Adare Manor Pilot (1.5d):** Install on Lynx server (coordinate MSM/Shaun Bowles). Config: irrigation_day_start=15:55, tz=Europe/Dublin. Dry-run → live → monitor 3–5 days.

---

## Phase 5 — QA & Hardening

### 5.1 — Promote staging to production

8 items deployed on stage, not yet in prod: #376 (Records API filters), #377 (Monthly irrigation data), #378 (Notification status API), #379 (Annual outflow data), #380 (ET data), #381 (Rainfall data), #382 (IoT DB structure), #383 (Daily totals script). Verify each, then deploy.

### 5.2 — Execute test cases

GitLab test cases: #345 (notifications), #346 (days watered modal, unblocked after Phase 1), #1192 (Water 2.0 general), #1193 (settings), #1224 (calendar behavior).

### 5.3 — Regression

Water source CRUD, reading CRUD (manual + connected), dashboard cards/modals, settings (ET factor, notifications, reminder time), unit conversion (metric ↔ imperial).

---

## Phase 6 — Vendor Strategy *(planning only)*

**6.1** Document standard vendor onboarding pattern based on Phase 3 unified table + Phase 4 Lynx pattern: config per vendor, data normalization (units/timestamps/granularity), Back Office UI.

**6.2** Prioritize Shayp / Masgrau / others by customer demand.

---

## Timeline

```
WK 1  ║ Phase 0: Bug fixes
WK 2  ║ Phase 1: DB + Model + Scheduler (1.1–1.3)
WK 3  ║ Phase 1: Notification APIs (1.4) | Phase 2: API docs → hand to mobile team (2.4)
WK 4  ║ Phase 1: Calendar + FE + graph (1.5–1.7) | Phase 2: Push service (2.1)
WK 5  ║ Phase 2: Mobile endpoints (2.2–2.3) | Phase 3: Design + migration (3.1–3.2)
WK 6  ║ Phase 3: Data + APIs + cron (3.3–3.6) | Phase 4: API contract (4.A1)
WK 7  ║ Phase 4: Cloud API (4.A2–A3) ║ Agent DB+reconciliation (4.B1–B2)
WK 8  ║ Phase 4: Health+data (4.A4–A5) ║ Agent push+CLI+package (4.B3–B5)
WK 9  ║ Phase 4: Integration test (4.C1)
WK 10 ║ Phase 4: Adare Manor pilot (4.C2) | Phase 5: Staging promotion (5.1)
WK 11 ║ Phase 5: Test cases + regression (5.2–5.3) | Phase 6: Vendor strategy
```

## Dependencies

```
Phase 0 → Phase 1 → Phase 2 → Mobile team builds app
               ↓
          Phase 3 → Phase 4 → Phase 5 → Phase 6
```

**Mobile critical path:** Phase 1 (Step 1.4) → Phase 2 → Mobile team → Mobile QA
