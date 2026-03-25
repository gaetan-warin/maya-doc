# Water Epic 341 — Production Migration Plan

**Created:** 2026-03-25 | **Verified against:** staging replica (`maya-prod`, batch 438)
**Source branch:** `feature/253/water-2-phase-2` (core-2.0) + `feature/water-device` (back-office)

---

## Summary

7 migrations must run on production. 34 water-related migrations are already deployed (batches 32-402).
No Lynx or irrigation notification tables exist yet in production.

---

## Pre-flight Checks

```sql
-- 1. Confirm current migration state
SELECT MAX(batch) AS last_batch, COUNT(*) AS total FROM migrations;
-- Expected: batch ~438, total ~714

-- 2. Confirm base tables exist
SHOW TABLES LIKE 'water_sources';
SHOW TABLES LIKE 'connected_water_meter_devices';
SHOW TABLES LIKE 'water_readings';
-- All three must exist

-- 3. Confirm target tables do NOT exist yet
SHOW TABLES LIKE 'irrigation_notifications';
SHOW TABLES LIKE 'lynx_%';
-- Must return empty
```

---

## Migrations to Run (in order)

### 1. Irrigation Notifications Table

**File:** `2026_01_18_000100_create_irrigation_notifications_table.php`

| Detail | Value |
|--------|-------|
| Action | CREATE TABLE `irrigation_notifications` |
| Purpose | Stores irrigation confirmed/denied status for the Days Watered calendar |
| Dependencies | FK to `water_sources` (exists), FK to `user` (exists) |
| Rollback | `DROP TABLE irrigation_notifications` |

```bash
php artisan migrate --path=database/migrations/2026_01_18_000100_create_irrigation_notifications_table.php
```

**Verify:**
```sql
SHOW CREATE TABLE irrigation_notifications;
-- Check: columns water_source_id, irrigation_date, irrigation_status, responded_by_user_id
-- Check: unique index on (water_source_id, irrigation_date)
```

---

### 2. Lynx Club Configs Table

**File:** `2026_03_17_100001_create_lynx_club_configs_table.php`

| Detail | Value |
|--------|-------|
| Action | CREATE TABLE `lynx_club_configs` |
| Purpose | Per-club configuration for Toro Lynx integration |
| Dependencies | None |
| Rollback | `DROP TABLE lynx_club_configs` |

```bash
php artisan migrate --path=database/migrations/2026_03_17_100001_create_lynx_club_configs_table.php
```

---

### 3. Lynx Water Records Table

**File:** `2026_03_17_100002_create_lynx_water_records_table.php`

| Detail | Value |
|--------|-------|
| Action | CREATE TABLE `lynx_water_records` |
| Purpose | Staging table for zone-level daily irrigation data from Lynx sync agent |
| Dependencies | FK to `lynx_club_configs` (migration #2), FK to `water_readings` (exists) |
| Rollback | `DROP TABLE lynx_water_records` |

```bash
php artisan migrate --path=database/migrations/2026_03_17_100002_create_lynx_water_records_table.php
```

---

### 4. Lynx Sync Logs Table

**File:** `2026_03_17_100003_create_lynx_sync_logs_table.php`

| Detail | Value |
|--------|-------|
| Action | CREATE TABLE `lynx_sync_logs` |
| Purpose | Immutable audit trail of every Lynx sync attempt |
| Dependencies | FK to `lynx_club_configs` (migration #2) |
| Rollback | `DROP TABLE lynx_sync_logs` |

```bash
php artisan migrate --path=database/migrations/2026_03_17_100003_create_lynx_sync_logs_table.php
```

---

### 5. Add `is_lynx_record` to Water Readings

**File:** `2026_03_17_100004_add_is_lynx_record_to_water_readings_table.php`

| Detail | Value |
|--------|-------|
| Action | ALTER TABLE `water_readings` ADD `is_lynx_record` BOOLEAN DEFAULT FALSE + index |
| Purpose | Distinguish Lynx records from connected meter records on the Water Page |
| Dependencies | `water_readings` table (exists) |
| Rollback | DROP COLUMN + DROP INDEX |

```bash
php artisan migrate --path=database/migrations/2026_03_17_100004_add_is_lynx_record_to_water_readings_table.php
```

**Verify:**
```sql
SHOW COLUMNS FROM water_readings LIKE 'is_lynx_record';
-- Check: tinyint(1), default 0, after is_connected_device_record
```

---

### 6. Add `site_id` to Lynx Club Configs

**File:** `2026_03_17_100005_add_site_id_to_lynx_club_configs_table.php`

| Detail | Value |
|--------|-------|
| Action | ALTER TABLE `lynx_club_configs` ADD `site_id` CHAR(16) NULLABLE + index |
| Purpose | Links club to a site for Water Page site-filtered views |
| Dependencies | `lynx_club_configs` (migration #2) |
| Rollback | DROP COLUMN + DROP INDEX |

```bash
php artisan migrate --path=database/migrations/2026_03_17_100005_add_site_id_to_lynx_club_configs_table.php
```

---

### 7. Add `api_token` to Connected Water Meter Devices

**File:** `2026_03_20_160000_add_api_token_to_connected_water_meter_devices_table.php`

| Detail | Value |
|--------|-------|
| Action | ALTER TABLE `connected_water_meter_devices` ADD `api_token` TEXT NULLABLE |
| Purpose | Stores encrypted bearer token for external source API (e.g. Shayp) |
| Dependencies | `connected_water_meter_devices` table (exists) |
| Rollback | DROP COLUMN |

```bash
php artisan migrate --path=database/migrations/2026_03_20_160000_add_api_token_to_connected_water_meter_devices_table.php
```

**Verify:**
```sql
SHOW COLUMNS FROM connected_water_meter_devices LIKE 'api_token';
-- Check: text, nullable, after water_source_id
```

---

## Run All at Once (alternative)

If running all 7 sequentially via Laravel:

```bash
php artisan migrate
```

This will pick up all pending migrations in date order. However, note that **other pending migrations** on the branch (e.g. `drop_users_table`) will also run. Prefer explicit `--path` execution listed above.

---

## Post-migration Verification

```sql
-- 1. Confirm all 7 ran
SELECT migration, batch FROM migrations
WHERE migration LIKE '%irrigation_notification%'
   OR migration LIKE '%lynx%'
   OR migration LIKE '%api_token_to_connected%'
ORDER BY id;
-- Expected: 7 rows

-- 2. Confirm new tables exist
SHOW TABLES LIKE 'irrigation_notifications';
SHOW TABLES LIKE 'lynx_club_configs';
SHOW TABLES LIKE 'lynx_water_records';
SHOW TABLES LIKE 'lynx_sync_logs';
-- All 4 must exist

-- 3. Confirm altered columns
SHOW COLUMNS FROM water_readings LIKE 'is_lynx_record';
SHOW COLUMNS FROM lynx_club_configs LIKE 'site_id';
SHOW COLUMNS FROM connected_water_meter_devices LIKE 'api_token';
-- All 3 must exist
```

---

## Manual Steps After Migrations

### Masgrau Devices (via back-office instead of seeder)

Create 3 Masgrau connected devices manually from the back-office UI (`/water-devices`):

| Name | Device Reference ID | Tenant | Source Type | Site |
|------|---------------------|--------|-------------|------|
| Masgrau PG1 - Nord | `masgrau-infinitum-pg1` | Infinitum Living | outflow | Infinitum Living site |
| Masgrau PG2 - Centre | `masgrau-infinitum-pg2` | Infinitum Living | outflow | Infinitum Living site |
| Masgrau PG3 - Sud | `masgrau-infinitum-pg3` | Infinitum Living | outflow | Infinitum Living site |

---

## Excluded Migrations

| Migration | Reason |
|-----------|--------|
| `2026_01_18_000101_drop_users_table` | Not water-related |
| `2026_03_13_000001_seed_masgrau_water_sources_and_devices` | Will be done manually via back-office |

---

## Rollback Plan

### Full rollback (reverse order)

```bash
# 1. Drop api_token column
php artisan migrate:rollback --path=database/migrations/2026_03_20_160000_add_api_token_to_connected_water_meter_devices_table.php

# 2. Rollback Lynx migrations (5 in reverse — cascading deletes clean up all Lynx data)
php artisan migrate:rollback --path=database/migrations/2026_03_17_100005_add_site_id_to_lynx_club_configs_table.php
php artisan migrate:rollback --path=database/migrations/2026_03_17_100004_add_is_lynx_record_to_water_readings_table.php
php artisan migrate:rollback --path=database/migrations/2026_03_17_100003_create_lynx_sync_logs_table.php
php artisan migrate:rollback --path=database/migrations/2026_03_17_100002_create_lynx_water_records_table.php
php artisan migrate:rollback --path=database/migrations/2026_03_17_100001_create_lynx_club_configs_table.php

# 3. Drop irrigation_notifications
php artisan migrate:rollback --path=database/migrations/2026_01_18_000100_create_irrigation_notifications_table.php
```

### Partial rollback (Lynx only)

If only Lynx needs to be rolled back, run steps 1-2 above. Irrigation notifications and api_token can remain safely — they have no Lynx dependency.
