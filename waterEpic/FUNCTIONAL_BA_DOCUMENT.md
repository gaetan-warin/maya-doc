# Water Page 2.0 — Functional Business Analysis

**Epic:** 253 — Water Page 2.0
**Date:** 2026-03-03
**Status:** In Development (partially implemented)
**Stakeholders:** Golf club greenkeepers, course managers, Maya product team, mobile team, MSM (Toro partner)

---

## 1. Business Context

### What is Maya?

Maya is a cloud-based course management platform used by golf clubs worldwide. It helps greenkeeping teams manage turf, irrigation, weather, agronomy, and daily operations from a single dashboard — on web and mobile.

### What is the Water Page?

The **Water Insight Page** is a module within Maya that gives greenkeepers a centralized view of their water usage, irrigation history, weather impact, and water budget. It replaces fragmented manual tracking (spreadsheets, paper logs, separate irrigation system reports) with a single source of truth.

### Why does it matter?

- **Regulatory pressure:** Many regions require golf clubs to report water usage and stay within annual allowances. Exceeding limits risks fines or license revocation.
- **Cost control:** Water is one of the largest operational expenses for golf courses. Visibility into consumption patterns drives savings.
- **Agronomic quality:** Over-watering and under-watering both damage turf. Data-driven irrigation decisions improve course quality.
- **Sustainability reporting:** Clubs increasingly report environmental metrics to members, governing bodies, and ESG frameworks.

### Who uses it?

| User | Role | Primary Need |
|------|------|-------------|
| **Greenkeeper** | Day-to-day operator | Log irrigation, respond to daily notifications, enter manual meter readings |
| **Course Manager / Head Greenkeeper** | Oversight & planning | Monitor budget, review irrigation history, configure notification settings |
| **Club General Manager** | Executive oversight | Annual water budget reporting, regulatory compliance |
| **Maya Admin (Back Office)** | System configuration | Configure connected meters, manage Lynx club configs, monitor sync health |

---

## 2. Product Vision

### Current State

The Water Page is partially built. The dashboard and basic water management work, but several critical capabilities are missing or incomplete:

| Capability | Status | User Impact |
|---|---|---|
| View water consumption dashboard | Working | Users can see current metrics |
| Log manual water readings | Working (bugs exist) | Users can enter data but encounter errors on update/delete |
| View irrigation calendar | Partially working | Calendar displays but status changes don't persist |
| Receive daily irrigation notifications | Not working | Frontend built, backend doesn't exist — notifications never generate |
| Respond to notifications (confirm/deny irrigation) | Not working | UI exists but calls non-existent APIs |
| Mobile irrigation alerts | Not implemented | Mobile team blocked — no APIs to integrate |
| Connected meter auto-readings | Basic model only | Hourly ingestion exists, unified structure incomplete |
| Toro Lynx integration | Not started | Adare Manor pilot committed, zero code exists |

### Target State

A greenkeeper's daily workflow should look like this:

1. **Morning:** Receive a push notification on mobile — _"You have 3 irrigation zones pending for today"_
2. **Review:** Open Maya (web or mobile), see which outflows need attention, with a suggested water volume pre-calculated based on their annual budget and recent conditions
3. **Respond:** Confirm or deny irrigation per zone, optionally entering the actual consumption value
4. **Automatic:** For clubs with Toro Lynx, readings flow in automatically — no manual entry needed. The system reconciles actual vs scheduled irrigation.
5. **Monitor:** Course manager checks the calendar for irrigation history, the budget card for remaining allowance, and the dashboard for environmental conditions (ET, rainfall, soil moisture)

---

## 3. Core Modules

### 3.1 Water Dashboard

**Purpose:** At-a-glance view of the club's water health.

**What the user sees:**

Six metric cards arranged in a responsive grid:

| Card | Shows | Data Source |
|------|-------|------------|
| **ET (Evapotranspiration)** | Last 24h ET, adjusted ET (with correction factor), total ET since last irrigation, tomorrow's forecast | Weather API |
| **Rainfall** | Last 24h rainfall, total since last irrigation, 24h forecast with probability | Weather API |
| **Water Usage** | Last 24h consumption, monthly total, monthly trend (up/down %), forecast | Water readings |
| **Days Watered** | Days irrigated this month, total this year, average daily consumption | Irrigation logs |
| **Site Conditions** | Air temp, soil temp, soil moisture (current + min/max) | Sensor data |
| **Water Budget** | Year-to-date consumption vs annual allowance, progress bar, comparison to last year | Water readings + settings |

**User interactions:**
- Click any card to open a detailed modal with historical charts
- Budget card shows color-coded progress: gray (<60%), yellow (60-80%), red (>80%)
- Budget card shows "Over by X" in red if annual allowance exceeded

---

### 3.2 Irrigation Notifications

**Purpose:** Daily prompts to record or confirm irrigation activity per outflow (water source).

**Business rules:**

1. **Generation:** Every day, the system checks each tenant's active outflows. For each outflow where today matches the irrigation schedule (daily, every other day, specific weekdays, or once per week), a "pending" notification is created.

2. **Suggested value:** Each notification includes a pre-calculated suggested consumption value based on:
   - Annual water allowance for this outflow
   - Remaining irrigation days in the period
   - ET correction factor (configured in settings)
   - Recent actual consumption patterns

3. **User response:** The greenkeeper can:
   - **Confirm** (irrigated) — optionally entering the actual consumption value
   - **Deny** (did not irrigate) — records that irrigation was skipped
   - **Bulk respond** — select multiple notifications, confirm or deny all at once, with optional bulk value distribution across selected outflows

4. **Notification lifecycle:**
   - `pending` → waiting for user response
   - `confirmed` → irrigation happened (value recorded)
   - `denied` → irrigation did not happen

5. **Display:** The web UI shows notifications for the past 7 days. Older notifications remain in the database as permanent daily logs.

**What the user sees:**

- **Notification cards** in the top section of the Water Page, grouped by outflow
- Each card shows: outflow name, date, suggested value, min/max range
- Expandable card: enter actual consumption value, add notes
- Bulk action bar: select multiple, confirm/deny with value distribution panel

**Mobile:**
- Push notification at the user's preferred reminder time (configured in settings)
- Notification list in the mobile app
- Quick respond from the app (confirm/deny + optional value)

---

### 3.3 Days Watered Calendar

**Purpose:** Visual monthly calendar showing irrigation history per outflow.

**What the user sees:**

- Month view calendar with navigation (previous/next month)
- Dropdown filter to select specific outflows or "All"
- Color-coded dots on each date:
  - **Green** = Irrigation confirmed
  - **Light blue** = Irrigation planned (pending notification)
  - Additional statuses possible (4-color logic per GitLab #1230)
- Hover tooltip showing total consumption for that day
- Total "days watered" count for the visible month

**User interactions:**

- Click a date to open a side panel
- Side panel shows outflow status for that date
- User can change status (confirm/deny irrigation) directly from the calendar
- Status changes create or update the corresponding `irrigation_daily_log` record

**Business rule:** Calendar status changes and notification responses share the same data model. Confirming irrigation from the calendar is identical to confirming a notification — both update the same `irrigation_daily_logs` row. One row per outflow per date (enforced by unique constraint).

---

### 3.4 Water Readings (Manual Entry)

**Purpose:** Greenkeepers manually log water meter readings or daily consumption values.

**Two measurement modes:**
1. **Meter reading** — user enters the accumulated meter value; system calculates consumption as (current - previous reading)
2. **Daily consumption** — user enters the volume used that day directly

**What the user sees:**

- Data management table showing all readings for a source
- Add new reading: select source, enter value, select date, add notes
- Edit existing reading: update value or notes
- Delete reading

**Known issues (Phase 0 bugs):**
- Cannot delete readings (#2254) — cascade logic fails
- Cannot update readings (#2258) — validation rejects valid edits
- Previous reading value shown incorrectly (#2263) — calculation or display bug
- Calendar total consumption incorrect (#2283) — aggregation bug

---

### 3.5 Connected Water Meters (IoT)

**Purpose:** Automatic water consumption data from physical IoT meters installed on water sources.

**Current state:**
- Device model exists (serial number + water source link)
- Hourly records ingestion API exists (`POST /water/hourly-records`)
- Daily aggregation cron job rolls hourly data into daily readings

**What needs to happen:**
- Unified table structure to support multiple data providers (Shayp, Masgrau, Lynx, others)
- Clear distinction in the UI between IoT-sourced readings and manually entered readings
- Pluggable architecture: adding a new meter vendor should not require code changes beyond a config entry

---

### 3.6 Toro Lynx Connector

**Purpose:** Automatic daily water usage sync from Toro Lynx irrigation control systems.

**Business context:**
- Toro Lynx is the most widely used irrigation system on golf courses
- Clubs want Lynx data in Maya automatically — no manual entry
- **Adare Manor** (Ireland) is the pilot customer, with a target of end of April 2026
- This is a **commercial module** — installation fee per club, scalable to 100+ clubs

**How it works (from the user's perspective):**

1. Maya installs a small program (the "Lynx Agent") on the club's Lynx server
2. Every day, the agent reads what actually happened (how much water each zone used)
3. The data automatically appears in Maya's Water Page — charts, calendar, budget, everything
4. The greenkeeper sees one clean number per zone per day. They never see "scheduled vs actual" — that's internal logic

**What the user sees differently:**
- Water readings appear automatically (tagged as connected device data)
- Each Lynx "zone" (e.g., Hole 1 Green, Hole 3 Fairway) maps to a Maya outflow
- No manual entry needed for clubs with Lynx
- Dashboard, calendar, and budget all populate automatically

**Behind the scenes (not visible to users):**

The system reconciles two data sources from the Lynx database:
- **Actuals** (what the sprinklers actually ran) — primary source, most accurate
- **Scheduled** (what was planned to run) — fallback when actuals are missing (e.g., during overseed periods)

This reconciliation produces one authoritative record per zone per day. The user never knows which source was used.

**Monitoring (Back Office):**
- Club health dashboard: green (syncing normally), yellow (>24h since last sync), red (>48h)
- Sync log viewer: last 30 syncs with record counts and error details
- Slack alerts when a club misses a sync for >26 hours

---

### 3.7 Water Settings

**Purpose:** Per-tenant configuration for the water module.

**User-configurable settings:**

| Setting | Description | Where |
|---------|------------|-------|
| **ET Correction Factor** | Percentage (0-100%) applied to ET values to adjust for local conditions | Tenant settings |
| **Mobile Notifications** | Enable/disable push notifications for irrigation reminders | Tenant settings |
| **Preferred Reminder Time** | Time of day to receive push notifications (e.g., 07:00) | Tenant settings |
| **Irrigation Period** | Start and end dates for the irrigation season (e.g., Apr 1 – Oct 31) | Per outflow |
| **Irrigation Frequency** | How often irrigation occurs: daily, every other day, specific weekdays, once per week | Per outflow |
| **Irrigation Weekdays** | Which days of the week (if frequency = specific weekdays) | Per outflow |
| **Annual Water Allowance** | Budget limit for the year per source | Per outflow |
| **Measurement Type** | Meter reading vs daily consumption | Per source |
| **Meter Unit** | Cubic meters or 100-cubic-feet | Per source |

---

## 4. User Journeys

### Journey 1: Daily Irrigation Management (Manual Club)

**Actor:** Greenkeeper at a club without connected meters or Lynx

```
1. 07:00 — Receives push notification: "You have 4 pending irrigation zones for today"
2. Opens Maya mobile app
3. Sees 4 notification cards (one per active outflow)
4. For Hole 1 Green: confirms irrigation, enters 12.5 m³
5. For Hole 3 Fairway: confirms irrigation, enters 8.2 m³
6. For Putting Green: denies irrigation (rain overnight, no need)
7. For Practice Area: confirms irrigation, accepts suggested value (10.0 m³)
8. Later, course manager opens web dashboard:
   - Water Usage card shows today's total consumption
   - Days Watered calendar shows green dots for confirmed zones
   - Budget card updates remaining allowance
```

### Journey 2: Automated Irrigation Monitoring (Lynx Club)

**Actor:** Course manager at Adare Manor (Lynx connected)

```
1. 06:00 — Lynx Agent runs automatically on the club's server
2. Agent queries what actually happened overnight (sprinkler run times)
3. Agent pushes data to Maya: 18 zones, each with total/auto/manual volume
4. Data appears automatically in Maya:
   - Dashboard cards update with latest consumption
   - Calendar shows green dots for irrigated zones
   - Budget card adjusts remaining allowance
5. Course manager reviews the Water Page:
   - Sees all 18 holes' irrigation data without manual entry
   - Notices Hole 7 Fairway used 2x normal volume → investigates a possible leak
   - Checks budget: 45% used, 55% remaining → on track
```

### Journey 3: Calendar Review & Correction

**Actor:** Head greenkeeper reviewing the week

```
1. Opens Water Page → clicks "Days Watered" card
2. Calendar shows this week's irrigation history
3. Filters to "Hole 5 Green" outflow only
4. Notices Wednesday shows as "pending" — greenkeeper forgot to confirm
5. Clicks Wednesday → side panel opens
6. Selects "Confirmed" → enters 9.8 m³ from memory
7. Wednesday dot turns green
8. Scrolls to see monthly total: 18 days watered, avg 11.2 m³/day
```

### Journey 4: Budget Monitoring

**Actor:** Club general manager preparing for board meeting

```
1. Opens Water Page → clicks "Water Budget" card
2. Sees progress bar at 72% (yellow) — 7 months into the year
3. Compared to last year: 5% less consumption at the same point
4. Notes: "We're tracking well, Lynx automation helped reduce over-watering"
5. Exports annual report for the board presentation
```

---

## 5. Data Model (Simplified)

```
Tenant (Golf Club)
  └── TenantWaterSettings (ET factor, notification prefs, reminder time)
  └── WaterSource[] (inflows and outflows)
        ├── SourceWaterSettings (irrigation period, frequency, weekdays, budget)
        ├── WaterReading[] (manual entries + connected device records)
        │     └── WaterSiteConsumption[] (distributed consumption per outflow)
        ├── IrrigationDailyLog[] (one per outflow per date)
        │     status: pending | confirmed | denied
        │     consumption_value, suggested_value
        │     source: schedule (auto-generated) | manual (user-created from calendar)
        └── ConnectedWaterMeterDevice[] (IoT meters linked to this source)
              └── ConnectedWaterMeterHourlyRecord[] (raw hourly data)

LynxClubConfig (per Lynx-connected club)
  ├── LynxWaterRecord[] (zone-day records from Lynx agent)
  └── LynxSyncLog[] (audit trail of syncs)
```

**Key relationship:** Both manual readings and Lynx records flow into `WaterReading` (the unified table). The dashboard, calendar, budget, and charts all read from `WaterReading`. The data source (manual, IoT meter, Lynx) is tracked but doesn't affect how the data is displayed.

---

## 6. Open Business Questions

These items need business input before or during implementation:

| # | Question | Impact | Status |
|---|----------|--------|--------|
| **BQ1** | What should happen when the scheduler generates a notification for a past date (e.g., system was down for 2 days)? Options: auto-confirm, skip, or create as "late pending" for the user to review. | Affects notification generation logic (Phase 1.3) | Needs decision |
| **BQ2** | Should legacy/pre-migration tenants receive notifications? Some clubs have old data but haven't been onboarded to Water 2.0 settings. | Affects notification filtering (Phase 1.3) | Needs decision |
| **BQ3** | When a user confirms irrigation via the calendar AND a notification exists for the same date/source, which input takes precedence? (Currently: same row, last write wins) | Affects calendar + notification interaction (Phase 1.5) | Needs decision |
| **BQ4** | What is the exact formula for `suggested_value`? Specifically: what data inputs, what weights, what edge cases (first day of season, no history, budget exhausted)? | Core business logic of the notification system (Phase 1.3) | Needs specification |
| **BQ5** | How should Lynx zones map to Maya outflows? Auto-create a WaterSource per zone on first sync, or require manual mapping in Back Office? | Affects Lynx onboarding UX (Phase 4.A5) | Needs decision |
| **BQ6** | Should the mobile notification show per-outflow detail or just a summary count? | Affects push notification content (Phase 2.1) | Needs decision |

---

## 7. Success Criteria

| Criterion | Measurement |
|-----------|-------------|
| Greenkeepers can manage daily irrigation via notifications | >80% of notifications responded to within 24h (post-launch metric) |
| No manual data entry for Lynx clubs | Adare Manor operates 30+ days without manual water readings |
| Water budget tracking is accurate | Budget calculations match manual audit within 2% |
| Mobile notifications drive engagement | Push notification open rate >50% |
| System reliability | Lynx sync success rate >98%, <1h mean time to recover from missed sync |
| Known bugs resolved | All 4 Phase 0 bugs closed and verified in production |

---

## 8. Phased Delivery

| Phase | What Users Get | When |
|-------|---------------|------|
| **Phase 0** | Bug fixes — delete/update readings work, correct values displayed | Week 1 |
| **Phase 1** | Working notifications + calendar persistence — the core daily workflow | Weeks 2-4 |
| **Phase 2** | Mobile push notifications + mobile API — irrigation on the go | Weeks 3-5 |
| **Phase 3** | Unified connected meter architecture — foundation for multi-vendor | Weeks 5-6 |
| **Phase 4** | Toro Lynx integration — automatic irrigation data for Adare Manor | Weeks 6-10 |
| **Phase 5** | QA + staging promotion — production readiness | Weeks 10-11 |
| **Phase 6** | Vendor strategy — roadmap for Shayp, Masgrau, others | Week 11+ |

---

## 9. Glossary

| Term | Definition |
|------|-----------|
| **Outflow** | A water source that distributes water (e.g., "Hole 1 Green sprinkler system"). Opposite of inflow (water supply). |
| **Inflow** | A water source that supplies water (e.g., borehole, mains supply, lake). |
| **ET (Evapotranspiration)** | The amount of water lost from soil and plants to the atmosphere. Higher ET = more irrigation needed. |
| **ET Correction Factor** | A percentage adjustment applied to raw ET data to account for local microclimate conditions. |
| **Irrigation Period** | The date range during which irrigation is active (e.g., April–October). Outside this period, no notifications are generated. |
| **Irrigation Day (Lynx)** | A non-calendar day used by Toro Lynx, typically spanning 3:55 PM to 3:55 PM. Water usage is grouped by irrigation day, not midnight-to-midnight. |
| **Zone (Lynx)** | A grouping of sprinkler stations in the Lynx system, identified by an area code (e.g., `1GR` = Hole 1 Green). Maps to a Maya outflow. |
| **Station (Lynx)** | An individual sprinkler head in the Lynx system (e.g., `1GR2` = second station on Hole 1 Green). |
| **Reconciliation** | The process of merging actual irrigation data (what sprinklers ran) with scheduled data (what was planned) to produce one authoritative record per zone per day. |
| **Backfill** | Importing historical data from the Lynx `water_use` table for months before the agent was installed, so the dashboard isn't empty on day one. |
| **Connected Device Record** | A water reading that was automatically generated by a connected meter or Lynx sync, as opposed to manual user entry. |
| **Tenant** | A golf club/organization in Maya. All data is tenant-scoped for multi-tenancy. |

---

_This document describes what the Water Page does and why. For technical implementation details, see `IMPLEMENTATION_PLAN.md`. For current codebase state, see `GAP_ANALYSIS_REPORT.md`._
