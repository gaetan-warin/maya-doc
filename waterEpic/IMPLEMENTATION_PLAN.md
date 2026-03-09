# Water Epic — Implementation Plan

**Date:** 2026-03-06  
**Based on:** `doc/waterEpic/GAP_ANALYSIS_REPORT.md`  
**Goal:** finish the missing work **without rebuilding what already exists**  
**Critical path:** Phase 1 → Phase 2 → Phase 3 → Phase 5 → Phase 6 → Phase 7 → Phase 8

---

## Already Verified in Code (Do Not Rebuild)

- [x] Water routes are registered under `api/v2`
- [x] Water source CRUD exists
- [x] Tenant/source water settings APIs exist
- [x] Water reading CRUD exists
- [x] Water 2.0 dashboard page, cards, and modals exist
- [x] `GET /water/calendar` exists
- [x] Connected-meter hourly ingestion exists
- [x] Connected-meter daily aggregation command exists
- [x] Shared Expo push infrastructure and `/api/v2/device-register` exist

---

## Phase 1 — Stabilize Existing Water Functionality

**Goal:** make the current Water 2.0 path reliable before adding new modules.

- [x] **1.1** Reproduce and fix `#2254` — delete water reading failure
- [x] **1.2** Reproduce and fix `#2258` — update water reading failure (+ wrapped update in DB::transaction, fixed strict-type cast on delete rollback)
- [x] **1.3** Reproduce and fix `#2263` — previous meter-reading value shown incorrectly
- [x] **1.4** Reproduce and fix `#2283` — calendar total consumption inconsistency (fixed: calendar query now uses `water_readings.consumption_value` directly instead of `water_site_consumption` to avoid rounding errors from site distribution and missing records)
- [x] **1.5** Add water-specific regression tests for the four fixes above (11 unit tests in `tests/Unit/Water/WaterReadingServiceTest.php`)
- [x] **1.6** Reclassify `#2259` and `#2268` — **Updated assessment:**
  - `#2259` (notifications not generating): Notification backend is **fully implemented** (model, migration, service, job, command, controller, routes). Issue is likely operational — scheduler not running or migration not applied in target environment. Reclassify as **ops/config issue**, not blocked by missing code.
  - `#2268` (calendar status change error): `PUT /water/calendar/sources/{sourceId}/status` endpoint does NOT exist. Reclassify as **blocked by Phase 4** (calendar write path).

**Definition of done**

- Delete / update / previous-reading / total-consumption flows work locally
- Water-specific backend tests exist for the fixed cases

---

## Phase 2 — Build the Notification Data Model

**Goal:** introduce the missing backend foundation that both notifications and calendar persistence depend on.

- [ ] **2.1** Create `irrigation_daily_logs` migration
- [ ] **2.2** Create model, repository, interface, and service binding
- [ ] **2.3** Define final status model and transitions:
  - `planned`
  - `pending`
  - `confirmed`
  - `denied`
  - `unknown`
  - `irrigated` (derived from actual reading)
- [ ] **2.4** Decide and document how manual water readings affect pending notifications
- [ ] **2.5** Implement suggested value calculation service
- [ ] **2.6** Implement min/max validation support for notification entry
- [ ] **2.7** Build daily notification generation scheduler/command

**Definition of done**

- Daily-log table exists
- Status rules are documented in code and docs
- Scheduler can generate pending items for configured outflows

---

## Phase 3 — Implement Notification APIs and Wire the Existing FE

**Goal:** make the Water 2.0 notification card actually work.

- [ ] **3.1** Create `GET /water/irrigation-notifications`
- [ ] **3.2** Create `PATCH /water/irrigation-notifications/{id}`
- [ ] **3.3** Create `POST /water/irrigation-notifications/batch-update`
- [ ] **3.4** Match the FE payload shape expected by `useWaterNotifications.js`
- [ ] **3.5** Wire the existing notification card against real responses
- [ ] **3.6** Validate bulk-confirm, bulk-deny, and single-confirm flows
- [ ] **3.7** Add backend tests for list / single update / batch update

**Definition of done**

- Notification card loads successfully on Water 2.0
- Single and bulk actions persist
- No notification-related 404s remain in Water 2.0

---

## Phase 4 — Complete Calendar Persistence

**Goal:** make calendar interactions real, consistent, and locked when actual readings exist.

- [ ] **4.1** Create `PUT /water/calendar/sources/{sourceId}/status`
- [ ] **4.2** Wire `useDateClickHandler.js` to the real API (remove the inline TODO path)
- [ ] **4.3** Keep “green / actual reading exists” days non-editable
- [ ] **4.4** Update calendar GET logic so it merges:
  - actual readings
  - planned schedule
  - confirmed notification states
  - denied notification states
  - unknown stale items
- [ ] **4.5** Re-test “All outflows” aggregation rules in the FE
- [ ] **4.6** Add backend tests for status update + merged calendar output

**Definition of done**

- Side panel saves successfully
- Inline day click saves successfully
- Calendar read output can return the full 4-color / full-status experience

---

## Phase 5 — Finish Connected Meter Reporting

**Goal:** make connected-meter data trustworthy and clearly distinguishable from manual data.

- [ ] **5.1** Update connected-meter daily aggregation so created `water_readings` are stored with `is_connected_device_record = true`
- [ ] **5.2** Verify the “Connected Meter Logs” tab only shows connected-generated readings
- [ ] **5.3** Verify manual logs tab excludes connected-generated readings
- [ ] **5.4** Decide whether the provider/unified-table work (`#427`-`#431`) is still required now that the current model exists
- [ ] **5.5** If still required, implement the provider/unified-table migration sequence
- [ ] **5.6** Document how `connected_water_meter_devices` are created/managed until there is a dedicated UI/API
- [ ] **5.7** Add tests around aggregation + connected/manual filtering

**Definition of done**

- Connected-generated daily readings are visibly and queryably distinct
- Read-only connected logs view is trustworthy

---

## Phase 6 — Add Water-Specific Mobile APIs

**Goal:** reuse the existing push infrastructure to unblock mobile water flows.

- [ ] **6.1** Create water irrigation push job/command using the existing `PushNotificationService`
- [ ] **6.2** Create mobile water notification list endpoint
- [ ] **6.3** Create mobile water submit/confirm endpoint
- [ ] **6.4** Respect tenant reminder time + user push enablement
- [ ] **6.5** Write mobile-facing API documentation with payload examples

**Definition of done**

- Mobile team has endpoints and payload docs
- Water notifications can be pushed without introducing a new push provider

---

## Phase 7 — Migrate NiFi Vendor Integrations to the Python Orchestrator

**Goal:** move the existing Shayp and Masgrau integrations out of NiFi and into the Python orchestrator with behavior parity.

- [ ] **7.1** Document the current Shayp NiFi flow and payload mapping
- [ ] **7.2** Document the current Masgrau NiFi flow and payload mapping
- [ ] **7.3** Define the Python orchestrator contract for vendor ingestion
- [ ] **7.4** Implement Shayp flow in the Python orchestrator
- [ ] **7.5** Implement Masgrau flow in the Python orchestrator
- [ ] **7.6** Validate output parity between NiFi and Python orchestrator
- [ ] **7.7** Define cutover and rollback procedure for each vendor
- [ ] **7.8** Document how future vendors plug into the orchestrator without Water Page code changes

**Definition of done**

- Shayp and Masgrau no longer depend on NiFi
- Python orchestrator produces equivalent normalized output
- Vendor cutover steps are documented and testable

---

## Phase 8 — Build Toro Lynx

**Goal:** build the greenfield Lynx module only after Water 2.0 core flows are stable.

### 8A — Cloud Ingest (Laravel)

- [ ] **8.A1** Create `lynx_club_configs`
- [ ] **8.A2** Create `lynx_water_records`
- [ ] **8.A3** Create `lynx_sync_logs`
- [ ] **8.A4** Add API-key auth / middleware
- [ ] **8.A5** Add `POST /api/v2/lynx/sync`
- [ ] **8.A6** Map Lynx zone data into Maya outflows / `water_readings`
- [ ] **8.A7** Add sync-log and health-monitoring endpoints/UI contract

### 8B — Agent

- [ ] **8.B1** Create separate `maya-lynx-agent` repo/project
- [ ] **8.B2** Add SQL Server query layer
- [ ] **8.B3** Add reconciliation logic (`actual` first, `scheduled` fallback)
- [ ] **8.B4** Add HTTPS push client
- [ ] **8.B5** Add CLI / config / scheduling / logging
- [ ] **8.B6** Package for Windows deployment

### 8C — Rollout

- [ ] **8.C1** Add end-to-end test path
- [ ] **8.C2** Pilot at Adare Manor
- [ ] **8.C3** Monitor first live syncs and fix onboarding issues

---

## Phase 9 — QA, Cleanup, and Release Readiness

**Goal:** close the loop and avoid handing the next developer another ambiguous state.

- [ ] **9.1** Add water-specific automated tests for all new APIs
- [ ] **9.2** Run a focused Water 2.0 regression pass:
  - source CRUD
  - reading CRUD
  - dashboard cards
  - calendar
  - notifications
  - connected meter logs
- [ ] **9.3** Verify which “stage” issues are already present in code vs still undocumented
- [ ] **9.4** Decide whether legacy Water v1 (`/water/graph-data`) stays in scope
- [ ] **9.5** If legacy Water v1 is out of scope, remove it from the critical path and document it as cleanup work

---

## Recommended Next 10 Tasks

If work starts now, do these in order:

- [x] Reproduce and fix `#2254`
- [x] Reproduce and fix `#2258`
- [x] Reproduce and fix `#2263`
- [ ] Reproduce and fix `#2283`
- [ ] Create `irrigation_daily_logs`
- [ ] Build notification scheduler
- [ ] Build notification list endpoint
- [ ] Build notification single-update endpoint
- [ ] Build notification batch-update endpoint
- [ ] Build calendar status update endpoint

---

## Out of Critical Path (Do Later or Only If Needed)

- [ ] Legacy `GET /water/graph-data` support for old Water v1 components
- [ ] Lynx fleet-management enhancements beyond pilot scope

---

## Bottom Line

The work left is **not** “build Water 2.0 from scratch”.  
The work left is:

1. fix current bugs  
2. add notification persistence  
3. add calendar writes  
4. finish connected-meter distinction  
5. expose water flows to mobile  
6. migrate Shayp and Masgrau from NiFi to the Python orchestrator  
7. build Lynx

That is the sequence that gives the safest path to finishing the project after handover.
