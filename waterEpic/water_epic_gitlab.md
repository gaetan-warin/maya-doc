Epic 341 - Water Update : Connected meter and Lynx integration
Open
  Epic created 18 hours ago by Valentine Godin
Epic
Epic 341 - Water Update : Connected meter and Lynx integration
Water Page 2.0 + Lynx Integration — March 2026
Target: Prod release end of March 2026.

Objective
Verify & fix the existing Water Page (built by previous team)
Strip out all irrigation planning and notification features
Add Toro Lynx as a new water data source and integration
Ship by end of March
Part 1: Verify & Fix Existing Water Page
The previous team built these. Verify and confirm they work. Fix anything missing.

1.1 Insight Cards (Top Right Section)
Six cards on the Water Page dashboard:

Card	Shows	Details
ET	ET last 24h, total since last irrigation, ET tomorrow	If correction factor ≠ 100% in settings, show both raw ET₀ and adjusted ET. Ensure ET modal uses bar chart.
Rainfall	Last 24h, since last irrigation, +24h forecast
Site Conditions	Air temp, soil temp, soil moisture — each with now/min/max (last 24h)
Water Usage	Last 24h total, this month total + trend vs last month, forecast this month	If no recent data: "Last reading: X days ago". Trend: ↑ red, ↓ green, = grey dash
Days Watered	Days this month, total this year, avg water/day + trend vs last month	Same trend indicators
Water Budget	Total this year vs annual allowance + progress bar, vs last year (saved/overused)	Progress bar: grey <60%, yellow 60-80%, red >80%
1.2 Card Modals
Each card opens a modal with detailed charts and filters.

Water Usage Modal:

Bar chart with tabs: Outflows (default) / Inflows
Range selector: 14 days (default), 30, 90, custom
View toggle: Daily (default) / Cumulative / Hourly (only if connected meter exists for selected outflow)
Pump/outflow multi-select with stacked bars per source
Connected meters show a small icon (e.g., 📶) next to their name
If mixed selection (some connected, some not) and Hourly selected: show validation message "Hourly data not available for [pump names]", only show hourly for connected outflows
Use bar chart (not line graph) for daily/cumulative views
Days Watered Modal:

Calendar view for current month (navigable to past months)
Outflow picker when multiple outflows exist
Dot legend — two states only:
🟢 Green = irrigation record exists for that day (manual entry, connected meter, or Lynx data)
🔴 Red = no data received, no data entry
Click on a date to toggle between green ↔️ red
If a water record already exists for that day+outflow (from connected meter or Lynx), user cannot change the status from the calendar — avoids conflicts with real data
Total irrigated days shown at top of modal and on the card
Water Budget Modal:

Full-year line chart (Jan–Dec), X = month, Y = total usage (m³)
Year selector: multi-year overlay for comparison
Outflow multi-select
View toggle: Monthly (default) / Cumulative
ET Modal:

Bar chart, range: 7 days (default), 30, 90, custom
Daily / Cumulative toggle
Shows ET₀ and adjusted ET if correction factor ≠ 100%
Rainfall Modal:

Chart (line or bar), range: 7 days (default), 30, 90, custom
Daily / Cumulative toggle
1.3 Water Settings
Per-outflow configuration:

Setting	Default	Purpose
Annual water allowance	0 (neutral)	Feeds Water Budget card
Irrigation period (from/to)	Jan–Dec	Filters dashboard data. If data exists outside selected range, show warning
ET correction factor (%)	100%	Adjusts ET display. Example: 85% → ET 5.2mm becomes 4.4mm
Removed from settings (no longer needed):

Default irrigation frequency (was for notification scheduling)
Mobile app notifications toggle (no more irrigation prompts)
Preferred reminder time (no more reminders)
1.4 Water Records Table (Bottom Section)
Feature	Detail
Tabs	Manual Readings / Connected Meter Logs
Columns	Date, Water Flow (pump/outflow name), Measurement (value + unit), Source icon, Comments, Edit, Delete
Source icons	💧 manual daily consumption / 📟 connected meter reading / 🔗 irrigation system (Lynx)
Filters	Date range, pump name, reading type — using standard UI components
Pagination	10 per page
1.5 Connected Water Meter (Shayp / Masgrau)
Feature	Detail
Data storage	Hourly inflow/outflow readings table (created by us)
Daily totals	Auto-computed from hourly records via cron
Source indicator	Manual vs automated readings clearly distinguished
Hourly view	Enables "Hourly" toggle in Water Usage modal for that outflow
Part 2: Remove Irrigation Planning & Notifications
Strip out entirely. This was the most complex part of Epic 253 and is to be removed.

Remove from the UI:

Top left section: daily irrigation notification cards ("Did you irrigate yesterday?")
Inline notification expansion / confirmation flow
Bulk mark yes/no controls
Light blue "planned irrigation" dots from Days Watered calendar
Irrigation frequency setting from Water Settings (keep in codebase for potential future use, remove from current release UI)
Mobile notification settings
Remove / don't build backend:

Notification scheduler/generator
Notification APIs (fetch, mark, bulk, submit)
Suggested value calculation service
Min/max range service
Mobile push notification service
Daily irrigation logs table
If any of this code was partially built by the previous team, remove it from this release. Keep the previous branch for records.

Part 3: Add Toro Lynx Integration (suggested structure, TBC)
3.1 What Lynx Is
Toro Lynx is an on-premise irrigation control system at golf clubs. It runs on a local SQL Server. We need to pull water usage data from it into Maya's Water Page.

First customer: Adare Manor (Ireland). This is the pilot.

3.2 Ingest API (Laravel / Core 2.0)
Build a new endpoint that receives daily water data from each club:

POST /api/v2/lynx/sync
Header: X-Lynx-Api-Key: <per-club-key>
Request payload:

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
data_source: actual (from real satellite data), scheduled (from planned schedule), or backfill (from historical table).

Response:

{
  "sync_id": "uuid",
  "status": "success",
  "accepted": 18,
  "rejected": 0,
  "errors": []
}
Auth: Per-club API key. Stored as bcrypt hash in DB. Displayed once on generation.

3.3 Data Model (suggestion, tbc)
Three new tables:

lynx_club_configs

club_id (FK to tenant)
api_key_hash
unit (m3 / gallons)
timezone
irrigation_day_start (time, e.g., "15:55")
last_sync_at
lynx_water_records

club_id
zone_code (e.g., "1GR")
zone_name (e.g., "Hole 1 Green")
irrigation_date
total_volume
auto_volume
manual_volume
data_source (actual / scheduled / backfill)
station_count
synced_at
Unique constraint: (club_id, zone_code, irrigation_date) — upsert on re-sync
lynx_sync_logs

sync_id
club_id
synced_at
status (success / partial / failed)
records_accepted
records_rejected
error_detail
3.4 Club Configuration (Back Office — Pilot Scope)
For Adare Manor pilot, keep it simple:

Admin form to configure a Lynx club: select tenant, set unit/timezone/irrigation day start
Generate API key (show once, store hash)
Health monitoring dashboard, sync log viewer, Slack alerts → minimal MVP to be operationally viable for CS team. Full version post-pilot.

3.5 Water Page — Displaying Lynx Data
Lynx zones appear as outflows in all Water Page views. One zone = one outflow, mapped as site + area + hole (e.g., "1GR" = Hole 1 Green).

View	How Lynx Data Appears
Water Usage card	Lynx zones included in totals. Daily data only (no hourly).
Water Usage modal	Lynx zones appear in outflow/pump selector. "Hourly" toggle hidden for Lynx outflows. Daily and cumulative work normally.
Days Watered calendar	🟢 green dot if Lynx reported irrigation that day. Per-zone view via outflow picker.
Water Budget	Lynx volume counts toward annual allowance.
Water Records table	Lynx records appear in "Connected Meter Logs" tab with 🔗 irrigation system source icon.
New capability: Per-hole/zone breakdown. Lynx gives us data per zone (Hole 1 Green, Hole 3 Fairway, etc.) — this is granularity Maya didn't have before. Display in Water Usage modal when a Lynx outflow is selected.

3.6 Python Sync Agent (Separate Repo: maya-lynx-agent)
Installed at each club on a machine that can reach the Lynx SQL Server. Runs daily, pushes data to Maya over HTTPS.

Tech: Python 3.11+, packaged as Windows .exe (PyInstaller). Config via YAML file. Scheduled via Windows Task Scheduler (daily 06:00 local time).

What it does each run:

Connect to local Lynx SQL Server (lynx_main database)
Query water_use_upload — actuals from satellite faceplates (last 7 days, most accurate)
Query schedule_activity_download — confirmed scheduled irrigation (fallback)
Reconcile: For each zone, each irrigation day:
Actuals exist? → Use them (captures manual watering, rain holds, actual runtime)
No actuals? → Use scheduled data (confirms schedule reached the faceplate)
Neither? → No record (gap)
Aggregate: Station-level → zone-level by parsing station_descriptor area code ("1GR2" → zone "1GR")
Handle irrigation day boundary: Group by irrigation day (e.g., 3:55 PM → 3:55 PM for Adare), not calendar day
Calculate volumes: (duration_seconds / 60) * station_flow
Push JSON to Maya's ingest API over HTTPS (port 443, outbound only — no firewall changes)
Log everything (rotating file, 10MB × 5)
CLI flags: --dry-run (query + reconcile, don't push), --backfill (include historical water_use table), --verbose

Known limitations (from MSM — not bugs, just how Lynx works):

Overseed periods (~2x/year): continuous all-day watering prevents the upload trigger → no actuals → agent falls back to scheduled data automatically
7-day retention on water_use_upload → agent must run at least weekly (daily recommended)
Satellite comms every 2 hours on even hours; must complete ≥1 hour before irrigation day ends
Rain holds don't clear the scheduled download record — only water_use_upload reflects actual zero/partial runtime
Station tags can be renamed by customer — always join on SUID (System Unique ID) for reliable identification, use station_descriptor for display/grouping only
3.7 Important Context: Meter Data vs Irrigation System Data
Not needed for March delivery.

Connected meters (Shayp/Masgrau) measure actual pump consumption. Lynx measures theoretical irrigation system output. They are two views of the same water — not additive.

No client today has both. When one does in the future: do not sum them. Use one source per outflow for budgets. The delta between the two reveals leaks/inefficiency. V2 feature — Q2 2026.

Out of Scope
Item	Why
Irrigation planning / notifications / nudging	Cancelled — Lynx replaces it
Mobile push notifications	Cancelled with notifications
Four-color calendar (planned = light blue)	Simplified to two colors
Data source conflict prevention (Shayp + Lynx)	No dual-source clients exist today — V2 Q2 2026
Health monitoring dashboard / Slack alerts	Minimal MVP for CS team. Full dashboard V2.
Sync log viewer	Minimal MVP for CS team. Full viewer V2.
Agent MSI installer with GUI wizard	Manual install for Adare Manor pilot. Installer needed for April rollout (~10 additional Lynx clubs expected). To be finalised post prod release.
Acceptance Criteria
Water Page loads correctly with Shayp/Masgrau data
All 6 insight cards show correct values with proper trend indicators and colors
Each card modal opens with working filters, chart types, and outflow selectors
Days Watered calendar shows 🟢 green (irrigated) / 🔴 red (not irrigated) — no other colors
Water Budget progress bar uses correct color thresholds
Water Settings work per outflow (allowance, period, ET factor)
No irrigation planning or notification UI visible anywhere
Lynx ingest API accepts zone-day data with per-club API key auth
Lynx zones appear as outflows in Water Usage card, modal, Days Watered, Budget, and Records table
Per-hole/zone breakdown visible when Lynx outflow is selected
Python agent queries Lynx SQL Server, reconciles actual vs scheduled, and pushes clean data to Maya
Agent handles irrigation day boundary correctly
Water Records table shows Lynx records alongside manual and meter records with distinct source icon