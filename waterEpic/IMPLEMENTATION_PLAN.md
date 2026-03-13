# Water Epic — Implementation Plan

**Date:** 2026-03-10 (updated) | **~30–33 dev days / ~7 weeks** (1 dev) | **~5 weeks** (2 devs)
**Ref:** `GAP_ANALYSIS_REPORT.md`, Epic 341 (GitLab) | **Critical path:** Phase 0 → 1 → 3 → 4

---

## Scope Change (March 2026)

Epic 341 replaces Epic 253. Major scope changes:
- **CANCELLED:** Irrigation notifications & planning (entire module)
- **CANCELLED:** Mobile push notifications & mobile APIs
- **CANCELLED:** 4-color calendar (simplified to 2 colors: green/red)
- **CANCELLED:** Notification-related settings (frequency, mobile toggle, reminder time)
- **ADDED:** Toro Lynx integration as primary deliverable
- **Target:** Prod release end of March 2026

---

## Already Verified in Code (Do Not Rebuild)

### Decisions (before starting)
- [x] **D1** Lynx data model → **own `lynx_*` tables (decided)** — different granularity (daily/zone vs hourly/device), needs auto/manual volume split + sync audit trail. Both feed into `water_readings` for dashboard.
- [x] **D2** Lynx agent repo → **separate `maya-lynx-agent` (decided)** — deployed as a self-contained installer (`.exe` via PyInstaller) on customer's Windows machine at the golf club. Zero shared code with Laravel/Vue.
- [x] **D3** Adare Manor backfill depth → **configurable per club (decided)** — agent `--backfill --months N` flag pulls N months of history from Lynx `water_use` table, tagged as `data_source = 'backfill'`. Default 12 months.
- [x] **D4** Notifications → **CANCELLED** — Lynx replaces irrigation planning. No notification system needed.
- [x] **D5** Mobile push → **CANCELLED** — cancelled with notifications.
- [x] **D6** Calendar colors → **2 colors only** — green (irrigated) / red (no data). No light blue "planned" dots.

### Phase 0 — Verify & Fix Existing Water Page *(4–5 days, Week 1)*
- [x] **0.1** Fix bug #2254 — Cannot delete water records
- [x] **0.2** Fix bug #2258 — Cannot update water records
- [x] **0.3** Fix bug #2263 — Previous reading value wrong in message (fixed `getPreviousWaterReading()` in `WaterReadingRepository.php`)
- [x] **0.4** Fix bug #2283 — Calendar total consumption wrongly updated (downstream fix from #2263)
- [x] **0.5** Close #2259 as "won't fix" — notification-related, feature cancelled
- [x] **0.6** Close #2268 as "won't fix" — notification system cancelled
- [ ] **0.7** Verify 6 insight cards render correctly with proper trend indicators and colors
- [ ] **0.8** Verify all card modals (Water Usage, Days Watered, Budget, ET, Rainfall) with filters and chart types
- [ ] **0.9** Verify Water Settings per outflow (allowance, period, ET factor)
- [ ] **0.10** Verify Water Records table (Manual Readings / Connected Meter Logs tabs, filters, pagination)

### Phase 1 — Remove Notification UI & Simplify Calendar + UI Rework *(2–3 days, Week 2)*
- [x] **1.1** Remove notification cards from top-left section of Water Page UI *(done 2026-03-12)*
- [x] **1.2** Remove inline notification expansion / confirmation flow *(done 2026-03-12)*
- [x] **1.3** Remove bulk mark yes/no controls *(done 2026-03-12)*
- [ ] **1.4** Remove light blue "planned irrigation" dots from Days Watered calendar
- [ ] **1.5** Simplify Days Watered calendar to 2 colors: green (irrigated) / red (no data)
- [ ] **1.6** Remove irrigation frequency setting from Water Settings UI (keep code for future use)
- [ ] **1.7** Remove mobile notification settings from UI
- [ ] **1.8** Remove/disable FE store methods calling non-existent notification APIs (`fetchIrrigationNotifications`, `updateIrrigationNotification`, `batchUpdateIrrigationNotifications`)
- [ ] **1.9** Implement click-to-toggle on Days Watered calendar (green ↔ red), but block toggle if connected meter or Lynx data exists for that day+outflow
- [ ] **1.10** Fix `GET /water/graph-data` endpoint (water.js:118) — create in `WaterController` or redirect FE to existing `/water/usage`
- [x] **1.11** Rework Water Page UI to follow standardized Volt/Tailwind design system *(done 2026-03-12)*
  - Replaced CoreUI, Bootstrap, AppButton, AppDialog, scoped CSS/SCSS with Volt components and Tailwind
  - Migrated all 6 dashboard cards to match dashboard card pattern (hover:scale-105, group, bi-box-arrow-up-right arrow icon)
  - Replaced `text-gray-*` → `text-surface-*` with `dark:` variants across all cards
  - Replaced `text-green-*` → `text-emerald-*` for brand consistency
  - Replaced `AppDialog` → Volt `Dialog`, `AppButton` → Volt `Button`/`SecondaryButton`/`DangerButton`
  - Replaced custom tab buttons → `MContentTabs`/`MContentTab` in WaterBottomSection
  - Replaced `CRow`/`CCol`/`CFormSelect` → Tailwind grid + Volt `Select`/`DatePicker` in WaterSourceForm
  - Replaced Bootstrap `spinner-border` → `MayaLoader`, `form-check-input` → Volt `Checkbox` in NotificationCard
  - Replaced raw inputs/buttons → Volt equivalents in BulkConsumptionPanel
  - Removed all scoped CSS/SCSS from water components
  - Added `CardLoadingSkeleton` shared component for consistent loading states

### Phase 2 — Verify Connected Meters *(3–4 days, Weeks 2–3)*
- [ ] **2.1** Verify Shayp/Masgrau data loads correctly on Water Page
- [ ] **2.2** Verify IoT vs manual distinction in Water Records table (#384 — in review)
- [ ] **2.3** Verify daily aggregation cron works correctly (#383 — on staging)
- [ ] **2.4** Verify hourly view toggle in Water Usage modal for connected meter outflows
- [ ] **2.5** Verify source icons display correctly (manual / connected meter / irrigation system)
- [ ] **2.6** Promote staging items to production: #376, #377, #378, #379, #380, #381, #382, #383

### Phase 3 — Toro Lynx Connector *(16.5 days, Weeks 3–7)*
- [ ] **3.A1** Define API contract + create DB migrations (1d)
- [ ] **3.A2** Build club config + API key management (1.5d)
- [ ] **3.A3** Build ingest API endpoint `POST /api/v2/lynx/sync` (1.5d)
- [ ] **3.A4** Build health monitoring — minimal MVP for CS team (1.5d)
- [ ] **3.A5** Wire Lynx data into Water Page dashboard (1.5d)
- [ ] **3.B1** Python: Lynx DB query layer via pyodbc (1.5d) *(parallel with A)*
- [ ] **3.B2** Python: Reconciliation engine (2d)
- [ ] **3.B3** Python: HTTPS push to Maya API (1d)
- [ ] **3.B4** Python: CLI + scheduling + logging (1d)
- [ ] **3.B5** Python: PyInstaller Windows package (1d)
- [ ] **3.C1** End-to-end integration test (1.5d)
- [ ] **3.C2** Adare Manor pilot deployment (1.5d)

### Phase 4 — QA & Hardening *(4 days, Weeks 7–8)*
- [ ] **4.1** Execute GitLab test cases: #1192 (Water 2.0 general), #1193 (settings), #1224 (calendar behavior)
- [ ] **4.2** Verify acceptance criteria from Epic 341
- [ ] **4.3** Full regression: source CRUD, reading CRUD (manual + connected + Lynx), dashboard, settings, unit conversion

### Phase 5 — Vendor Strategy *(planning only, post-release)*
- [ ] **5.1** Document vendor onboarding architecture based on Lynx pattern
- [ ] **5.2** Prioritize Shayp / Masgrau / others by customer demand

---

## Phase 1 — Stabilize Existing Water Functionality

- **Developers left** — no verbal handover, only codebase + documents
- **Routes confirmed** registered via `RouteServiceProvider.boot()` under `api/v2`
- **Lynx Connector** — Adare Manor pilot, deadline end of March 2026
- **Lynx data model** — own `lynx_*` tables (different granularity: daily zone-level vs hourly device-level)
- **Notifications cancelled** — Lynx replaces irrigation planning. Existing notification FE code to be removed.
- **Mobile cancelled** — no mobile push notifications or mobile-specific APIs in this release

---

## Phase 0 — Verify & Fix Existing Water Page

**Goal:** Fix what's broken, verify what's working. No new features.

### Bug Fixes

| Bug | Issue | Investigation path |
|-----|-------|--------------------|
| #2254 | Can't delete water records | `WaterReadingController@destroy` → check `WaterReadingService.deleteWaterReading()` + `DeleteWaterSiteConsumption` listener cascade + FE store error handling |
| #2258 | Can't update water records | `WaterReadingController@update` → check `WaterReadingService.updateWaterReading()` validation (meter reading can't go below previous) |
| #2263 | Wrong previous reading value | Check `WaterReadingService.calculateWaterReadingValues()` or FE form "previous reading" hint |
| #2283 | Calendar total consumption wrong | Check `WaterUsageService.getIrrigationCalendar()` + `dashboardDataMapper.mapDaysWateredData()` in FE |
| #2259 | Calendar error on status change | **Close as "won't fix"** — notification feature cancelled by Epic 341 |
| #2268 | Notifications not generating | **Close as "won't fix"** — notification system cancelled by Epic 341 |

### Verification Checklist

Per Epic 341 acceptance criteria — verify each card, modal, setting, and table works correctly:

| Area | What to verify |
|------|---------------|
| ET Card | ET last 24h, total since last irrigation, ET tomorrow. If correction factor ≠ 100%, show both raw ET₀ and adjusted ET. Bar chart in modal. |
| Rainfall Card | Last 24h, since last irrigation, +24h forecast |
| Site Conditions Card | Air temp, soil temp, soil moisture — each with now/min/max (last 24h) |
| Water Usage Card | Last 24h total, this month total + trend vs last month, forecast. "Last reading: X days ago" if no recent data. Trend: ↑ red, ↓ green, = grey |
| Days Watered Card | Days this month, total this year, avg water/day + trend vs last month |
| Water Budget Card | Total this year vs annual allowance + progress bar (grey <60%, yellow 60-80%, red >80%), vs last year |
| Water Usage Modal | Bar chart, tabs (Outflows/Inflows), range selector (14d/30/90/custom), daily/cumulative/hourly toggle, multi-select outflows |
| Days Watered Modal | Calendar month view, outflow picker, green/red dots only |
| Water Budget Modal | Full-year line chart (Jan–Dec), year selector, outflow multi-select, monthly/cumulative toggle |
| ET Modal | Bar chart, range 7d/30/90/custom, daily/cumulative toggle |
| Rainfall Modal | Chart, range 7d/30/90/custom, daily/cumulative toggle |
| Settings | Annual allowance, irrigation period (from/to), ET correction factor per outflow |
| Records Table | Manual Readings / Connected Meter Logs tabs, date/pump/type filters, pagination (10/page), source icons |

---

## Phase 1 — Remove Notification UI & Simplify Calendar + UI Rework

**Goal:** Strip out cancelled features. Simplify Days Watered to 2-color system. Migrate UI to Volt/Tailwind design system.

### What to remove from UI
- ~~Top left section: daily irrigation notification cards ("Did you irrigate yesterday?")~~ **DONE 2026-03-12**
- ~~Inline notification expansion / confirmation flow~~ **DONE 2026-03-12**
- ~~Bulk mark yes/no controls~~ **DONE 2026-03-12**
- Light blue "planned irrigation" dots from Days Watered calendar
- Irrigation frequency setting from Water Settings (keep in codebase)
- Mobile notification settings

### UI Rework (DONE 2026-03-12)
Full migration of Water Page to standardized Volt/Tailwind design system:
- Removed NotificationCard from WaterTopSection, simplified grid to `grid-cols-1 sm:grid-cols-2 lg:grid-cols-3`
- All 6 dashboard cards now match dashboard pattern (group hover, scale, arrow icon for clickable cards)
- All colors migrated: `text-gray-*` → `text-surface-*`, `text-green-*` → `text-emerald-*`, `bg-gray-*` → `bg-surface-*`
- All CoreUI/Bootstrap components replaced with Volt equivalents (Dialog, Button, Select, Checkbox, etc.)
- Bottom section tabs migrated to `MContentTabs`/`MContentTab`
- All scoped CSS/SCSS removed, using Tailwind utilities only
- Files modified: water.vue, WaterTopSection, WaterBottomSection, WaterHeaderSection, WaterInsightCard, WaterReadingsTable, WaterSourceForm, WaterModalContainer, NotificationCard, BulkConsumptionPanel, ETCard, RainfallCard, WaterUsageCard, DaysWateredCard, SiteConditionsCard, WaterBudgetCard, CardLoadingSkeleton

### What to remove/disable in backend
- Don't build: notification scheduler, notification APIs, suggested value service, min/max range service, daily irrigation logs table
- Disable FE store methods that call non-existent notification endpoints (prevent silent errors)
- Close GitLab issues as "won't do": #348, #349, #350, #351, #352, #363, #364, #437, #370, #371, #372, #436, #1235

### Calendar simplification
- Days Watered calendar: **2 colors only**
  - Green = irrigation record exists (manual entry, connected meter, or Lynx data)
  - Red = no data received, no data entry
- Click on a date to toggle green ↔ red
- If water record exists from connected meter or Lynx for that day+outflow → user **cannot** toggle (prevents conflicts with real data)
- Total irrigated days shown at top of modal and on card

### Graph-data endpoint
FE store calls `GET /water/graph-data` (water.js:118) → either create in `WaterController` or redirect FE to existing `/water/usage`.

---

## Phase 2 — Verify Connected Meters

**Goal:** Ensure Shayp/Masgrau data works correctly. Promote staging items to prod.

### Verification

| Check | What | GitLab |
|-------|------|--------|
| Data loads | Shayp/Masgrau hourly records appear on Water Page | — |
| IoT vs Manual | Source distinction visible in Records table | #384 (in review) |
| Daily aggregation | `AggregateConnectedWaterMeterDailyReadings` cron works | #383 (staging) |
| Hourly view | "Hourly" toggle appears in Water Usage modal for connected meter outflows | — |
| Mixed selection | If mixed connected + manual outflows selected and "Hourly" chosen: show validation message | — |
| Source icons | Manual (💧) / Connected meter (📟) / Irrigation system (🔗) | — |

### Staging → Production Promotion

8 items on staging, verify and deploy:
- #376: Water Records API with Advanced Filters
- #377: Monthly Irrigation Data API
- #378: Update Notification Status API
- #379: Annual Outflow Data API
- #380: ET Data API
- #381: Rainfall Data API
- #382: Database Structure for IoT Readings
- #383: Daily Totals Calculation Script

---

## Phase 3 — Toro Lynx Connector

**Goal:** reuse the existing push infrastructure to unblock mobile water flows.

- [ ] **6.1** Create water irrigation push job/command using the existing `PushNotificationService`
- [ ] **6.2** Create mobile water notification list endpoint
- [ ] **6.3** Create mobile water submit/confirm endpoint
- [ ] **6.4** Respect tenant reminder time + user push enablement
- [ ] **6.5** Write mobile-facing API documentation with payload examples

**3.A1 — API Contract + Data Model (1d)**

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

**3.A2 — Club Config + API Key Management (1.5d)**

`LynxClubConfig` model. Back Office Vue page: list/create/edit clubs, generate API key (shown once, stored bcrypt). `X-Lynx-Api-Key` auth middleware.

**3.A3 — Ingest Endpoint (1.5d)**

`POST /api/v2/lynx/sync` — authenticate via API key → validate payload → upsert `lynx_water_records` → create/update `WaterReading` (with `is_connected_device_record = true`) → log to `lynx_sync_logs` → return accept/reject counts.

**3.A4 — Health Monitoring — Minimal MVP (1.5d)**

Daily scheduled job: flag clubs with no sync in >26h → Slack alert. Minimal Back Office visibility for CS team. Full dashboard deferred to V2.

**3.A5 — Water Page Integration (1.5d)**

Lynx zones appear as outflows in all Water Page views:

| View | How Lynx Data Appears |
|------|----------------------|
| Water Usage card | Lynx zones included in totals. Daily data only (no hourly). |
| Water Usage modal | Lynx zones in outflow selector. "Hourly" toggle hidden for Lynx outflows. Daily + cumulative work normally. |
| Days Watered calendar | Green dot if Lynx reported irrigation that day. Per-zone view via outflow picker. |
| Water Budget | Lynx volume counts toward annual allowance. |
| Water Records table | Lynx records in "Connected Meter Logs" tab with 🔗 irrigation system source icon. |

New capability: **per-hole/zone breakdown** when a Lynx outflow is selected.

Each zone maps to a `WaterSource` (outflow), auto-created on first sync or configured in club config.

### Workstream B — Sync Agent (Python, parallel with A)

**3.B1 — DB Query Layer (1.5d):** Project setup (`maya-lynx-agent`), SQL Server via pyodbc, query `water_use_upload` (actuals, 7-day window) + `schedule_activity_download` (scheduled fallback), join `station` on SUID, config from YAML.

**3.B2 — Reconciliation Engine (2d):** Per zone per irrigation day: actuals → scheduled → skip. Station→zone aggregation (parse `1GR2` → `1GR`). Irrigation day boundary (3:55 PM configurable). Volume = `(duration/60) × station_flow`. Manual = total - auto. Edge cases: overseed, zero-duration rain hold, re-runs (upsert).

**3.B3 — HTTPS Push (1d):** Build JSON matching A1 schema, POST with retry (3×, exponential backoff), handle responses + errors.

**3.B4 — CLI + Scheduling (1d):** Flags: `--config`, `--dry-run`, `--backfill`, `--verbose`. Exit codes: 0=success, 1=partial, 2=API fail, 3=DB fail, 4=config error. Rotating log (10MB×5). Windows Task Scheduler docs.

**3.B5 — Windows Package (1d):** PyInstaller → single `.exe`. Install to `C:\MayaLynx\`. Template `config.yaml`. Manual install for Adare Manor pilot. MSI installer deferred to April rollout (~10 additional clubs).

### Integration

**3.C1 — E2E Test (1.5d):** Mock Lynx DB → Agent → API → Water Page. Test: actuals only, scheduled fallback, mixed, gaps, overseed, rain hold, re-sync.

**3.C2 — Adare Manor Pilot (1.5d):** Install on Lynx server (coordinate MSM/Shaun Bowles). Config: irrigation_day_start=15:55, tz=Europe/Dublin. Dry-run → live → monitor 3–5 days.

---

## Phase 4 — QA & Hardening

### 4.1 — Execute test cases

GitLab test cases: #1192 (Water 2.0 general), #1193 (settings), #1224 (calendar behavior).

Note: #345 (notifications) and #346 (days watered modal with notification status) are **no longer applicable** — notification feature cancelled.

### 4.2 — Acceptance criteria (from Epic 341)

- [ ] Water Page loads correctly with Shayp/Masgrau data
- [ ] All 6 insight cards show correct values with proper trend indicators and colors
- [ ] Each card modal opens with working filters, chart types, and outflow selectors
- [ ] Days Watered calendar shows green (irrigated) / red (not irrigated) — no other colors
- [ ] Water Budget progress bar uses correct color thresholds
- [ ] Water Settings work per outflow (allowance, period, ET factor)
- [ ] No irrigation planning or notification UI visible anywhere
- [ ] Lynx ingest API accepts zone-day data with per-club API key auth
- [ ] Lynx zones appear as outflows in Water Usage card, modal, Days Watered, Budget, and Records table
- [ ] Per-hole/zone breakdown visible when Lynx outflow is selected
- [ ] Python agent queries Lynx SQL Server, reconciles actual vs scheduled, pushes clean data to Maya
- [ ] Agent handles irrigation day boundary correctly
- [ ] Water Records table shows Lynx records alongside manual and meter records with distinct source icon

### 4.3 — Regression

Water source CRUD, reading CRUD (manual + connected + Lynx), dashboard cards/modals, settings (ET factor, allowance, period), unit conversion (metric ↔ imperial).

---

## Phase 5 — Vendor Strategy *(planning only, post-release)*

**5.1** Document standard vendor onboarding pattern based on Lynx pattern: config per vendor, data normalization (units/timestamps/granularity), Back Office UI.

**5.2** Prioritize Shayp / Masgrau / others by customer demand.

---

## Out of Scope (per Epic 341)

| Item | Why |
|------|-----|
| Irrigation planning / notifications / nudging | Cancelled — Lynx replaces it |
| Mobile push notifications | Cancelled with notifications |
| Four-color calendar (planned = light blue) | Simplified to two colors |
| Data source conflict prevention (Shayp + Lynx) | No dual-source clients exist today — V2 Q2 2026 |
| Full health monitoring dashboard / Slack alerts | Minimal MVP for CS team. Full dashboard V2. |
| Full sync log viewer | Minimal MVP for CS team. Full viewer V2. |
| Agent MSI installer with GUI wizard | Manual install for pilot. Installer for April rollout. |

---

## Phase 9 — QA, Cleanup, and Release Readiness

```
WK 1  ║ Phase 0: Bug fixes (#2254, #2258, #2263, #2283) + verify existing Water Page
WK 2  ║ Phase 1: Remove notification UI + simplify calendar | Phase 2: Verify connected meters
WK 3  ║ Phase 2: Promote staging to prod | Phase 3: API contract + club config (3.A1–A2)
WK 4  ║ Phase 3: Cloud API (3.A3) ║ Agent DB + reconciliation (3.B1–B2)
WK 5  ║ Phase 3: Health + Water Page (3.A4–A5) ║ Agent push + CLI + package (3.B3–B5)
WK 6  ║ Phase 3: Integration test + pilot (3.C1–C2)
WK 7  ║ Phase 4: QA, acceptance criteria, regression
WK 8  ║ Buffer / Phase 5: Vendor strategy (planning only)
```

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

```
Phase 0 → Phase 1 → Phase 2
                       ↓
                  Phase 3 → Phase 4 → Phase 5
```

**Lynx critical path:** Phase 0 (bug fixes) → Phase 3 (Lynx connector) → Phase 4 (QA)

## GitLab Issues — Disposition Under Epic 341

| Action | Issues |
|--------|--------|
| **Fix** | #2254, #2258, #2263, #2283 |
| **Close as "won't fix"** | #2259, #2268 (notification-related) |
| **Close as "won't do"** | #348, #349, #350, #351, #352, #363, #364, #437, #370, #371, #372, #436, #439, #1230, #1235 (notification/mobile features cancelled) |
| **Verify & promote** | #376, #377, #378, #379, #380, #381, #382, #383 (on staging) |
| **Verify** | #384 (in review), #438 (testing), #440 (testing) |
| **Keep for QA** | #1192, #1193, #1224 |
| **No longer applicable** | #345, #346 (notification test cases) |
