# Water Epic 341 — Implementation Plan

**Last updated:** 2026-03-17 | **Target:** Prod release end of March 2026
**Ref:** Epic 341 (GitLab), `LYNX_COMPLETE_REFERENCE.md`, `GO_LIVE_PROCEDURE.md`

---

## Scope

| | |
|-|-|
| **CANCELLED** | Irrigation notifications & planning, mobile push, 4-color calendar, notification settings |
| **ADDED** | Toro Lynx integration (primary deliverable) |
| **ADDED** | Back office forms for all water integrations (replaces manual seed migrations) |

---

## Architecture Decisions (locked)

| # | Decision |
|---|----------|
| D1 | Lynx uses own `lynx_*` tables (daily/zone granularity vs hourly/device). Both feed `water_readings`. |
| D2 | Lynx agent is a separate repo `maya-lynx-agent`, deployed as `.exe` on customer Windows machine. |
| D3 | Backfill depth configurable per club via `--backfill --months N` flag. Default 12 months. |
| D4 | Connected meter devices (Masgrau/Shayp) managed via back office form — no more seed migrations. |
| D5 | `LynxDailyPromotionService` promotes `lynx_water_records` → `water_readings` (separate from ingest). |
| D6 | Laravel scheduler (`routes/console.php`) owns schedule declarations. DevOps owns running it. |
| D7 | Notifications cancelled. Lynx replaces irrigation planning. Calendar: 2 colors only (green/red). |
| D8 | Lynx zones are **always outflows** (`source_type = 'outflow'` hardcoded). No back office field for it. Masgrau/Shayp can be inflow or outflow — configurable. |
| D9 | Lynx record detection in Water Page via `is_lynx_record` boolean on `water_readings` — consistent with existing `is_connected_device_record` pattern. One migration to add the column. |
| D10 | `measurement_type` is not configurable in back office — derived from integration type. Masgrau → always `meter_reading` (Modbus cumulative registers). Shayp → always `daily_consumption` (API hourly deltas). Admin picks integration type; system hardcodes the rest. |

---

## Scheduled Commands (DevOps must confirm all three are running)

| Command | Schedule | Purpose |
|---------|----------|---------|
| `water:aggregate-connected-daily-readings` | Hourly | `connected_water_meter_hourly_records` → `water_readings` (Masgrau + Shayp) |
| `water:promote-lynx-daily-readings` | Daily 06:30 | `lynx_water_records` → `water_readings` (Lynx) |
| `lynx:check-sync-health` | Daily 08:00 | Flag clubs with no sync in >26h → Slack alert |

All declared in `routes/console.php`. DevOps to ensure `php artisan schedule:run` runs every minute in production.

---

## Delivery Timeline

```
Week 1   Phase 0: Bug fixes + verify existing Water Page (DONE)
Week 2   Phase 1: Remove notifications + simplify calendar + UI rework (DONE)
         Phase 2: Verify connected meters (mostly DONE — staging → prod pending)
Week 3   ← NOW → Phase 3.A: Back office forms | Phase 3.B: Lynx backend
Week 4   Phase 3.C: Water Page Lynx display | Phase 3.D: Health monitoring
         Phase 3.E: Python agent DB layer + reconciliation (parallel)
Week 5   Phase 3.E: Python agent push + CLI + package (parallel)
Week 6   Phase 3.F: E2E integration test + Adare Manor pilot
Week 7   Phase 4: QA, acceptance criteria, regression
```

---

## Phase 0 — Verify & Fix Existing Water Page ✅ DONE

- [x] **0.1** Fix bug #2254 — Cannot delete water records
- [x] **0.2** Fix bug #2258 — Cannot update water records
- [x] **0.3** Fix bug #2263 — Previous reading value wrong (`getPreviousWaterReading()` fixed)
- [x] **0.4** Fix bug #2283 — Calendar total consumption wrongly updated (downstream of #2263)
- [x] **0.5** Close #2259 as "won't fix" — notification-related, feature cancelled
- [x] **0.6** Close #2268 as "won't fix" — notification system cancelled
- [x] **0.7** Verify 6 insight cards — ET, Rainfall, SiteConditions, WaterUsage, DaysWatered, WaterBudget *(2026-03-13)*
- [x] **0.8** Verify all 5 card modals with filters and chart types *(2026-03-13)*
- [x] **0.9** Verify Water Settings per outflow — allowance, irrigation period, ET factor *(2026-03-13)*
- [x] **0.10** Verify Water Records table — tabs, filters, pagination, source icons *(2026-03-13)*

---

## Phase 1 — Remove Notifications + Simplify Calendar + UI Rework ✅ DONE

- [x] **1.1** Remove notification cards from Water Page top-left *(2026-03-12)*
- [x] **1.2** Remove inline notification expansion / confirmation flow *(2026-03-12)*
- [x] **1.3** Remove bulk mark yes/no controls *(2026-03-12)*
- [x] **1.4** Remove light blue "planned irrigation" dots from calendar *(2026-03-13)*
- [x] **1.5** Simplify calendar to 2 colors: green (irrigated) / red (no data) *(2026-03-13)*
- [x] **1.6** Hide irrigation frequency setting in Water Settings UI (keep code) *(2026-03-13)*
- [x] **1.7** Remove mobile notification settings — N/A, didn't exist *(2026-03-13)*
- [x] **1.8** Disable FE store methods calling non-existent notification APIs (no-op stubs) *(2026-03-13)*
- [x] **1.9** Implement 2-color calendar click-to-toggle — block toggle if connected meter or Lynx data exists *(2026-03-13)*
- [x] **1.10** Fix `GET /water/graph-data` endpoint — added `getWaterGraphData()` in WaterController *(2026-03-13)*
- [x] **1.11** Full UI rework to Volt/Tailwind design system *(2026-03-12)*

---

## Phase 2 — Verify Connected Meters (Shayp / Masgrau)

- [x] **2.1** Verify Shayp/Masgrau data loads on Water Page — 69 readings confirmed *(2026-03-13)*
- [x] **2.2** Verify IoT vs manual distinction in Water Records table *(2026-03-13)*
- [x] **2.3** Verify daily aggregation cron works — 67 readings created, 2 bugs fixed *(2026-03-13)*
- [x] **2.4** Verify hourly view toggle in Water Usage modal for connected meter outflows *(2026-03-13)*
- [x] **2.5** Verify source icons — speedometer (meter_reading), droplet (daily_consumption), wifi badge *(2026-03-13)*
- [ ] **2.6** Promote staging items to production: #376, #377, #378, #379, #380, #381, #382, #383

---

## Phase 3 — Toro Lynx Integration

### 3.0 — Verified (no work needed)

- [x] **3.0.1** Unique constraint on `connected_water_meter_hourly_records` — already declared in original CREATE TABLE migration (`cwmhr_device_datetime_unique`). No additional migration needed. *(verified 2026-03-17)*

---

### 3.A — Back Office: Water Integration Management (3d)

> **Dependency:** 3.B.1 migrations must be run before back office forms can be tested end-to-end. Build 3.B.1 first, then 3.A.

Replaces all manual seed migrations. Admins configure all integrations from the back office.

#### 3.A.1 — Back Office: Connected Meter Devices (Masgrau / Shayp) (1.5d)

**Backend** (`core-2.0`):

| File | What |
|------|------|
| `app/Http/Controllers/Water/ConnectedWaterMeterDeviceController.php` | `index, store, update, destroy` |
| `app/Http/Requests/Water/CreateConnectedWaterMeterDeviceRequest.php` | Validation |
| `app/Http/Requests/Water/UpdateConnectedWaterMeterDeviceRequest.php` | Validation |
| `app/Resources/Water/ConnectedWaterMeterDeviceResource.php` | JSON transformer |

Route in `routes/api.php`:
```php
Route::apiResource('water/connected-meter-devices', ConnectedWaterMeterDeviceController::class);
```

**Back Office Frontend** (`back-office`):

| File | What |
|------|------|
| `src/stores/connectedWaterMeter.ts` | `fetchDevices, createDevice, updateDevice, deleteDevice` |
| `src/pages/water/index.vue` | List all devices, filter by tenant, "Add Device" button |
| `src/components/water/ConnectedMeterDeviceForm.vue` | Modal form |

Form fields:
- **Water Source section:** tenant (select), name, `source_type` (inflow / outflow), site association (filtered by tenant)
- **Device section:** integration type (Masgrau / Shayp — drives `measurement_type` automatically), `device_reference_id`, status
- On save: creates `water_source` + `water_source_default_associations` + `connected_water_meter_device` in one `DB::transaction()`
- On delete device: keeps `water_source` intact (historical readings must not be lost)

**Masgrau seed migration transition:**
1. Build and deploy this back office form
2. Recreate the 3 Masgrau devices (PG1/PG2/PG3) via the form for Infinitum Living tenant
3. Verify existing `water_readings` still link correctly (they FK to `water_source_id`, not the device — no data loss)
4. Mark `2026_03_13_000001_seed_masgrau_water_sources_and_devices.php` as superseded in comments

- [x] Backend controller + request + service *(2026-03-17)*
  - `CreateConnectedMeterDeviceRequest.php` — validates tenant, name, source_type, integration_type, device_reference_id, site_id
  - `ConnectedMeterDeviceAdminService.php` — atomic create (water_source + device + associations), list, soft-delete
  - `ConnectedWaterMeterController.php` — index, store, destroy
  - Routes: `GET/POST /api/v2/water/connected-devices`, `DELETE /api/v2/water/connected-devices/{id}`
- [x] Back office store + page + form *(2026-03-17)*
  - `src/stores/connectedMeterDevice.ts` — fetchDevices, createDevice, deleteDevice
  - `src/pages/water/index.vue` — list table + create modal
  - `src/types/water.ts` — TypeScript interfaces
  - i18n keys added to en.yml, fr.yml, es.yml
- [ ] Masgrau 3 devices recreated via form, seed migration superseded

#### 3.A.2 — Back Office: Lynx Club Configuration (1.5d)

**Backend** (`core-2.0`):

| File | What |
|------|------|
| `app/Http/Controllers/Lynx/LynxClubConfigController.php` | `index, store, update, destroy, generateApiKey` |
| `app/Http/Requests/Lynx/CreateLynxClubConfigRequest.php` | Validation |
| `app/Http/Requests/Lynx/UpdateLynxClubConfigRequest.php` | Validation |
| `app/Services/Lynx/LynxApiKeyService.php` | `generate()` → returns plaintext once, stores bcrypt hash |
| `app/Resources/Lynx/LynxClubConfigResource.php` | JSON transformer |

Routes in `routes/api.php`:
```php
Route::apiResource('lynx/clubs', LynxClubConfigController::class);
Route::post('lynx/clubs/{id}/generate-key', [LynxClubConfigController::class, 'generateApiKey']);
```

**Back Office Frontend** (`back-office`):

| File | What |
|------|------|
| `src/stores/lynxClub.ts` | `fetchClubs, createClub, updateClub, deleteClub, generateApiKey` |
| `src/pages/water/lynx.vue` | List clubs: tenant name, last_sync_at, last_sync_status, API key status |
| `src/components/water/LynxClubForm.vue` | Modal form — 2 tabs: Configuration / Sync Logs |

Form fields (Configuration tab): tenant (select), `club_identifier`, unit (m3 / gallons), timezone (select), `irrigation_day_start` (time picker)
Note: **no `source_type` field** — Lynx zones are always outflows (D8).
Sync Logs tab: read-only table from `lynx_sync_logs` for that club.
Generate API Key: shows plaintext key once in modal with copy button — warns it won't be shown again.

- [x] Backend controller + requests + service + resource *(2026-03-17)*
  - `LynxClubConfigController.php` — index, store, update, destroy, generateApiKey, syncLogs
  - `CreateLynxClubConfigRequest.php`, `UpdateLynxClubConfigRequest.php`
  - `LynxApiKeyService.php` — generates `lynx_` prefixed random key, stores bcrypt hash
  - `LynxClubConfigResource.php` — JSON transformer (hides api_key_hash, exposes has_api_key)
  - Routes: `apiResource lynx/clubs`, `POST lynx/clubs/{id}/generate-key`, `GET lynx/clubs/{id}/sync-logs`
- [x] Back office store + page + form *(2026-03-17)*
  - `src/stores/lynxClub.ts` — CRUD + generateApiKey + fetchSyncLogs
  - `src/pages/water/lynx.vue` — list table + create/edit modals + API key display modal + sync logs modal
  - i18n keys added to all 3 locale files

---

### 3.B — Lynx Backend: Data Model + Ingest + Promotion (3.5d)

> **Build order: 3.B.1 → 3.B.2 → 3.B.3 → 3.B.4** (each step depends on the previous)

#### 3.B.1 — Migrations (0.5d)

| Migration | Table |
|-----------|-------|
| `2026_03_XX_create_lynx_club_configs_table.php` | `lynx_club_configs` — tenant_id, club_identifier (unique), api_key_hash, unit, timezone, irrigation_day_start, last_sync_at, last_sync_status |
| `2026_03_XX_create_lynx_water_records_table.php` | `lynx_water_records` — lynx_club_config_id (FK), zone_code, zone_name, irrigation_date, total_volume, auto_volume, manual_volume, data_source, station_count, sync_id, water_reading_id (FK nullable). UNIQUE: (lynx_club_config_id, zone_code, irrigation_date) |
| `2026_03_XX_create_lynx_sync_logs_table.php` | `lynx_sync_logs` — lynx_club_config_id (FK), sync_id, agent_version, records_received, records_accepted, records_rejected, status, error_details (json), duration_ms |
| `2026_03_XX_add_is_lynx_record_to_water_readings.php` | Add `is_lynx_record` boolean default false to `water_readings` — enables Lynx detection in Water Page (D9) |

- [x] 4 migrations created *(2026-03-17)*
  - `2026_03_17_100001_create_lynx_club_configs_table.php`
  - `2026_03_17_100002_create_lynx_water_records_table.php`
  - `2026_03_17_100003_create_lynx_sync_logs_table.php`
  - `2026_03_17_100004_add_is_lynx_record_to_water_readings_table.php`
- [x] `WaterReading` model updated with `is_lynx_record` field *(2026-03-17)*
- [x] Migrations run successfully on cloud dev DB (via core2 Docker container) *(2026-03-17)*
- [x] All 4 confirmed present in DB (`lynx_club_configs`, `lynx_water_records`, `lynx_sync_logs`, `is_lynx_record` column on `water_readings`) *(2026-03-17)*

#### 3.B.2 — Models + Repositories (0.5d)

| File | What |
|------|------|
| `app/Models/Lynx/LynxClubConfig.php` | Model |
| `app/Models/Lynx/LynxWaterRecord.php` | Model |
| `app/Models/Lynx/LynxSyncLog.php` | Model |
| `app/Enums/LynxDataSource.php` | `actual / scheduled / backfill` |
| `app/Enums/LynxSyncStatus.php` | `success / partial / failed` |
| `app/Interfaces/Lynx/LynxClubConfigRepositoryInterface.php` | Contract |
| `app/Interfaces/Lynx/LynxWaterRecordRepositoryInterface.php` | Contract |
| `app/Repositories/Lynx/LynxClubConfigRepository.php` | Implementation |
| `app/Repositories/Lynx/LynxWaterRecordRepository.php` | Implementation |

- [x] Models + enums + repositories created *(2026-03-17)*
  - `app/Enums/LynxDataSource.php`, `app/Enums/LynxSyncStatus.php`
  - `app/Models/Lynx/LynxClubConfig.php`, `LynxWaterRecord.php`, `LynxSyncLog.php`
  - `app/Interfaces/Lynx/LynxClubConfigRepositoryInterface.php`, `LynxWaterRecordRepositoryInterface.php`
  - `app/Repositories/Lynx/LynxClubConfigRepository.php`, `LynxWaterRecordRepository.php`
  - Bindings registered in `RepositoryServiceProvider.php`

#### 3.B.3 — Ingest Endpoint (1d)

`POST /api/v2/lynx/sync` — authenticated via `X-Lynx-Api-Key` header.

| File | What |
|------|------|
| `app/Http/Middleware/LynxApiKeyAuth.php` | Resolves club config from bcrypt-hashed key |
| `app/Http/Requests/Lynx/LynxSyncRequest.php` | Validates full payload |
| `app/Http/Controllers/Lynx/LynxSyncController.php` | Orchestrates ingest |
| `app/Services/Lynx/LynxIngestService.php` | Upserts `lynx_water_records`, writes `lynx_sync_logs`, updates `last_sync_at` on config |

**Unit conversion:** payload arrives in the club's configured unit (`m3` or `gallons`). `LynxIngestService` must normalize volumes to the tenant's preferred unit before writing `lynx_water_records`. Use `lynx_club_configs.unit` for the conversion.

Request payload:
```json
{
  "sync_id": "uuid",
  "club_identifier": "adare-manor",
  "agent_version": "1.0.0",
  "sync_date": "2026-03-15",
  "unit": "m3",
  "records": [{
    "zone_code": "1GR",
    "zone_name": "Hole 1 Green",
    "irrigation_date": "2026-03-14",
    "total_volume": 45.2,
    "auto_volume": 42.1,
    "manual_volume": 3.1,
    "data_source": "actual",
    "station_count": 8
  }]
}
```

Response:
```json
{ "sync_id": "uuid", "status": "success", "accepted": 18, "rejected": 0, "errors": [] }
```

Route in `routes/api.php`:
```php
Route::post('lynx/sync', [LynxSyncController::class, 'sync'])->middleware(LynxApiKeyAuth::class);
```

- [x] Middleware + request + controller + service *(2026-03-17)*
  - `LynxApiKeyAuth` — bcrypt-checks `X-Lynx-Api-Key` against `api_key_hash`, binds config to request
  - `LynxSyncRequest` — validates full payload including per-record fields
  - `LynxIngestService` — upserts records, converts units, writes sync log, updates config
  - `LynxSyncController` — thin orchestrator with error logging
- [x] Route registered in `routes/api/lynx.php` + `RouteServiceProvider` *(2026-03-17)*
- [x] Upsert on re-sync (UNIQUE constraint + `findByClubZoneDate` in repository)
- [x] Unit conversion applied before writing (gallons ↔ m3 in LynxIngestService)
- [x] Sync log written on every call (success and failure)

#### 3.B.4 — Promotion Service + Command (1d)

Runs daily, promotes unpromoted `lynx_water_records` → `water_readings`.

| File | What |
|------|------|
| `app/Services/Lynx/LynxDailyPromotionService.php` | Core logic |
| `app/Console/Commands/PromoteLynxDailyReadings.php` | Artisan command `water:promote-lynx-daily-readings` |

Service logic:
1. For each `lynx_club_config`, query `lynx_water_records` where `water_reading_id IS NULL`
2. For each `zone_code`: find or create `water_source` — **always `source_type = 'outflow'`**, linked to tenant via `lynx_club_configs.tenant_id`
3. On zone `water_source` creation: also create `water_source_default_associations` to link the source to the tenant's site (required for Water Page site filtering)
4. Call `WaterReadingService::createWaterReading()` with `is_lynx_record = true` and `is_connected_device_record = true`
5. **Timezone note:** `irrigation_date` comes from the agent already adjusted for irrigation day boundary — use it as-is, do not re-derive from UTC
6. Set `lynx_water_records.water_reading_id` = id of created reading

Add to `routes/console.php`:
```php
// Promote Lynx daily zone records to water readings — runs after agent pushes at 06:00
Schedule::command('water:promote-lynx-daily-readings')->dailyAt('06:30');
```

- [x] Service created — `app/Services/Lynx/LynxDailyPromotionService.php` *(2026-03-17)*
  - Loops all club configs → finds unpromoted lynx_water_records → promotes each
  - `findOrCreateWaterSource()` keyed on `(tenant_id, water_meter_id = zone_code)`
  - Water source created as `outflow` + `daily_consumption` (D8, D10)
  - `site_id` nullable on `lynx_club_configs` — site association created when set
  - Extra migration added: `2026_03_17_100005_add_site_id_to_lynx_club_configs_table.php` ✅ RAN
- [x] `water_source` auto-created per zone on first encounter (water_source_default_associations created when site_id is configured)
- [x] `is_lynx_record = true` + `is_connected_device_record = true` set on created `water_readings`
- [x] `irrigation_date` used as-is (no timezone re-derivation per plan note)
- [x] Command created — `app/Console/Commands/PromoteLynxDailyReadings.php` with `$signature = 'water:promote-lynx-daily-readings'`
- [x] Schedule added to `routes/console.php` — `dailyAt('06:30')` *(2026-03-17)*
- [x] DevOps notified — GO_LIVE_PROCEDURE.md step 2b updated with all 3 scheduled commands *(2026-03-17)*

---

### 3.C — Water Page: Display Lynx Data (1.5d)

Lynx zones land in `water_readings` via promotion service — most views work automatically.
Lynx records are identified by `is_lynx_record = true` on `water_readings` (D9). The `GET /water/readings` and `GET /water/sources` API responses must expose this flag.

Explicit frontend changes:

| Component | Change |
|-----------|--------|
| `WaterUsageModal` | Hide "Hourly" toggle when selected outflow has `is_lynx_record` readings |
| `DaysWateredCalendar` | Lock dots for days where `is_lynx_record = true` — extend existing connected meter lock logic |
| `WaterRecordsTable` | Show 🔗 icon for rows where `is_lynx_record = true` |
| `WaterUsageModal` | Per-zone breakdown panel when a Lynx outflow is selected |

- [x] `is_lynx_record` exposed in water readings API response — field is in `$fillable` + `$casts`, auto-serialized *(2026-03-17)*
- [x] Hourly toggle hidden for Lynx outflows — Lynx sources have no `connected_water_meter_device`, toggle already conditional *(2026-03-17)*
- [x] 🔗 icon in Water Records table — already implemented in `WaterReadingsTable.vue:101`, now uses proper `is_lynx_record` flag *(2026-03-17)*
- [x] Calendar dots locked for Lynx days — `useDaysWateredStatus.js` protects `irrigated` status (which Lynx readings produce) *(2026-03-17)*
- [x] `useWaterReadings.js` fixed: Lynx detection changed from notes-string-matching to `!!reading.is_lynx_record` flag; `can_edit`/`can_delete` disabled for Lynx records *(2026-03-17)*
- [ ] Per-zone breakdown in Water Usage modal (deferred to V2 — requires additional API endpoint to return per-zone data from `lynx_water_records`)

---

### 3.D — Health Monitoring — Minimal MVP (1d)

Daily job: flag clubs with no sync in >26h → Slack alert.
Back office visibility: `last_sync_at` + `last_sync_status` already shown in Lynx clubs list (3.A.2).
Full health dashboard deferred to V2.

| File | What |
|------|------|
| `app/Console/Commands/CheckLynxSyncHealth.php` | `lynx:check-sync-health` |
| `app/Services/Lynx/LynxHealthService.php` | Query stale clubs, send Slack alert |

Add to `routes/console.php`:
```php
Schedule::command('lynx:check-sync-health')->dailyAt('08:00');
```

- [x] Command + service created *(2026-03-17)*
  - `app/Console/Commands/CheckLynxSyncHealth.php` — `lynx:check-sync-health`
  - `app/Services/Lynx/LynxHealthService.php` — queries stale clubs (>26h since last sync), sends Slack webhook
  - Schedule added to `routes/console.php` — `dailyAt('08:00')`
- [x] Slack alert fires when club has no sync in >26h — via `config('services.slack.lynx_webhook_url')`
- [x] Alert includes club name, last_sync_at, last_sync_status

---

### 3.E — Python Sync Agent (separate repo `maya-lynx-agent`, parallel with 3.B–3.D)

#### 3.E.1 — DB Query Layer (1.5d)

```
maya-lynx-agent/
├── config.yaml                  (host, port, db, api_key, api_url, timezone, irrigation_day_start)
├── agent.py                     (entry point)
├── lynx/
│   ├── connection.py            (pyodbc SQL Server connection)
│   ├── queries.py               (water_use_upload + schedule_activity_download + station join on SUID)
│   └── models.py                (dataclasses: StationRecord, ZoneRecord)
```

- [ ] pyodbc connection configured from YAML
- [ ] `water_use_upload` query (7-day window, join `station` on SUID)
- [ ] `schedule_activity_download` query (fallback, join `station` on SUID)

#### 3.E.2 — Reconciliation Engine (2d)

```
lynx/
├── reconciler.py    (actuals vs scheduled per zone per irrigation day)
├── aggregator.py    (station → zone: "1GR2" → "1GR", volume = (duration/60) × station_flow)
└── boundary.py      (irrigation day boundary — configurable, default 15:55)
```

Logic per zone per irrigation day:
1. Has `water_use_upload` actuals? → use them (`data_source = "actual"`)
2. No actuals? → has `schedule_activity_download`? → use it (`data_source = "scheduled"`)
3. Neither? → no record (gap — don't invent data)

Edge cases: overseed (multiple downloads/day → sum all), rain hold (zero actual = valid, include it), re-sync (upsert via UNIQUE constraint), satellite failure (accept gap).

- [ ] All edge cases handled per `LYNX_COMPLETE_REFERENCE.md` section 4
- [ ] Irrigation day boundary correct (timestamps before `day_start` belong to previous day)
- [ ] `manual_volume = total_duration - auto_duration`

#### 3.E.3 — HTTPS Push (1d)

```
maya/
└── client.py    (POST /api/v2/lynx/sync, retry 3× exponential backoff)
```

- [ ] Payload matches API contract (3.B.3)
- [ ] Retry 3× with exponential backoff on 5xx
- [ ] 4xx errors logged and not retried (config/auth issue — needs human intervention)
- [ ] Full response body logged on failure

#### 3.E.4 — CLI + Logging (1d)

Flags: `--config <path>`, `--dry-run` (reconcile only, no push — outputs JSON to stdout), `--backfill` (include `water_use` historical table), `--verbose`
Exit codes: 0=success, 1=partial, 2=API fail, 3=DB fail, 4=config error
Rotating log: 10MB × 5 files at `C:\MayaLynx\logs\`

- [ ] All CLI flags implemented
- [ ] Rotating log configured
- [ ] `--dry-run` outputs reconciled payload JSON without pushing
- [ ] Exit codes documented in README

#### 3.E.5 — Windows Package (1d)

- PyInstaller → single `.exe`
- Install path: `C:\MayaLynx\`
- Template `config.yaml` bundled
- Windows Task Scheduler: daily 06:00 local time (agent runs before 06:30 promotion service)
- Manual install for Adare Manor pilot. MSI installer deferred to April (~10 additional clubs).

- [ ] `.exe` builds and runs on clean Windows machine
- [ ] `config.yaml` template documented
- [ ] Task Scheduler setup instructions written in README

---

### 3.F — Integration Test + Pilot (3d)

#### 3.F.1 — E2E Test (1.5d)

Mock Lynx DB → agent → API → promotion service → Water Page.

| Scenario | Expected outcome |
|----------|-----------------|
| Actuals only | `data_source = "actual"`, correct volumes |
| Scheduled fallback | `data_source = "scheduled"` when no actuals |
| Mixed zones (some actual, some scheduled) | Each zone independently sourced |
| Gap (no data either source) | No record created |
| Overseed (multiple downloads) | Sum of all download records |
| Rain hold | Zero actual included (`data_source = "actual"`, volume = 0) |
| Re-sync same day | Upsert — latest sync wins, no duplicate |

- [ ] All 7 scenarios pass `--dry-run`
- [ ] All 7 scenarios push correctly to staging and appear on Water Page

#### 3.F.2 — Adare Manor Pilot (1.5d)

- Coordinate with MSM / Shaun Bowles for access to Lynx server
- Configure: `irrigation_day_start=15:55`, `timezone=Europe/Dublin`, `club_identifier=adare-manor`
- Run `--dry-run` → review reconciled output with MSM before pushing
- Go live → monitor 3–5 days
- Verify zones appear as outflows on Water Page for Adare Manor tenant

- [ ] Dry-run output reviewed and approved by MSM
- [ ] Live push confirmed — sync log shows `status=success`
- [ ] Water Page shows Adare Manor zones as outflows with correct volumes
- [ ] No duplicate records after agent re-runs

---

## Phase 4 — QA & Hardening (4d)

### 4.1 — Execute test cases

GitLab: #1192 (Water 2.0 general), #1193 (settings), #1224 (calendar behavior)
Note: #345, #346 (notification test cases) — no longer applicable, feature cancelled.

### 4.2 — Acceptance criteria (Epic 341)

- [ ] Water Page loads correctly with Shayp/Masgrau data
- [ ] All 6 insight cards: correct values, trend indicators, colors
- [ ] Each card modal: working filters, chart types, outflow selectors
- [ ] Days Watered: green/red only — no other colors
- [ ] Water Budget progress bar: correct thresholds (grey <60%, yellow 60-80%, red >80%)
- [ ] Water Settings per outflow: allowance, period, ET factor
- [ ] No irrigation planning or notification UI visible anywhere
- [ ] Connected meter devices manageable from back office (no seed migrations needed)
- [ ] Lynx ingest API accepts zone-day data with per-club API key auth
- [ ] Lynx club configurable from back office: tenant, timezone, irrigation_day_start, API key
- [ ] Lynx zones appear as outflows in: Usage card, Usage modal, Days Watered, Budget, Records table
- [ ] Per-hole/zone breakdown visible when Lynx outflow selected in Usage modal
- [ ] Hourly toggle hidden for Lynx outflows
- [ ] Python agent: reconciles actual vs scheduled, correct irrigation day boundary, pushes to Maya
- [ ] Water Records table: Lynx records with 🔗 icon alongside manual and meter records
- [ ] `water:aggregate-connected-daily-readings` running hourly (DevOps confirmed)
- [ ] `water:promote-lynx-daily-readings` running daily at 06:30 (DevOps confirmed)
- [ ] `lynx:check-sync-health` running daily at 08:00 (DevOps confirmed)

### 4.3 — Regression

Water source CRUD, reading CRUD (manual + connected + Lynx), dashboard, calendar, settings (ET factor, allowance, period), unit conversion (m³ ↔ gallons / 100 ft³).

---

## Out of Scope (Epic 341)

| Item | Reason |
|------|--------|
| Irrigation planning / notifications | Cancelled — Lynx replaces it |
| Mobile push notifications | Cancelled |
| 4-color calendar | Simplified to 2 colors |
| Data source conflict (Shayp + Lynx same outflow) | No dual-source clients today — V2 Q2 2026 |
| Full health monitoring dashboard | Minimal Slack alert MVP only — full dashboard V2 |
| Full sync log viewer in back office | Sync Logs tab on club form only — full viewer V2 |
| Agent MSI installer with GUI | Manual install for pilot — MSI for April rollout |

---

## GitLab Issues Disposition

| Action | Issues |
|--------|--------|
| **Fixed** | #2254, #2258, #2263, #2283 |
| **Closed "won't fix"** | #2259, #2268 |
| **Closed "won't do"** | #348, #349, #350, #351, #352, #363, #364, #437, #370, #371, #372, #436, #439, #1230, #1235 |
| **Verify & promote to prod** | #376, #377, #378, #379, #380, #381, #382, #383 |
| **In review / testing** | #384, #438, #440 |
| **Keep for QA** | #1192, #1193, #1224 |
| **No longer applicable** | #345, #346 |
