# Database Field Reference — Accurate Descriptions from Codebase Analysis

> **Generated:** 2025-12-22  
> **Source:** Database schema query via Docker + code usage analysis  
> **Database:** maya-prod (MySQL 8.0.43)

---

## Task Table (39 columns)

### Core Identifiers

| Field | Type | Description (from code analysis) |
|-------|------|----------------------------------|
| `idtask` | BINARY(16) | Primary key (UUID) |
| `idgroup` | BINARY(16) | FK to group/course the task belongs to |
| `idsite` | BINARY(16) | FK to site where task is performed |
| `idaction` | BINARY(16) | FK to action table (mowing, aeration, etc.) |
| `iduser` | BINARY(16) | FK to user who created the task |
| `idstaff` | BINARY(16) | FK to assigned staff member |
| `idmachinery` | BINARY(16) | FK to assigned machinery |
| `idparent_task` | BINARY(16) | FK to parent task for sub-task hierarchies |

### Merge Key System (Legacy)

| Field | Type | Description |
|-------|------|-------------|
| `merge_key` | LONGTEXT | Concatenated identifier grouping cloned task rows for multi-location tasks. Generated from: siteIds + holeIds + modificationDate + actionId + timestamp |
| `merged_time` | TIMESTAMP | Timestamp when the `merge_key` was generated (used for migration/tracking) |

### Task Execution State

| Field | Type | Description |
|-------|------|-------------|
| `task_start` | TIMESTAMP | Actual start time (set when task begins on mobile) |
| `task_end` | TIMESTAMP | Actual end time (set when task completes on mobile) |
| `running` | TINYINT | 1 = task completed, 0 = not completed (legacy boolean) |
| `status` | TINYINT | Task status: 1=todo, 2=in_progress, 3=completed, 4=not_completed |

### Time Scheduling

| Field | Type | Description |
|-------|------|-------------|
| `estimate_time` | INT UNSIGNED | Estimated duration in **minutes** |
| `is_am` | TINYINT | 1 = Morning slot, 0 = not AM |
| `is_pm` | TINYINT | 1 = Afternoon slot, 0 = not PM |
| `is_over_time` | TINYINT(1) | 1 = Task is overtime work |
| `is_all` | TINYINT(1) | **All areas selected flag** - When true, task applies to all areas of the site, not specific selections |

### Recurrence

| Field | Type | Description |
|-------|------|-------------|
| `recurrent` | TINYINT | 1 = Task recurs on schedule |
| `recurrence_rule` | TEXT | iCal-style RRULE string (e.g., "FREQ=WEEKLY;BYDAY=MO,WE,FR") |
| `next_execution` | DATETIME | Next scheduled occurrence |
| `last_execution` | DATETIME | Most recent execution |

### Notifications

| Field | Type | Description |
|-------|------|-------------|
| `notification_via` | VARCHAR(100) | Notification channel. Default: `email`. Can be: email, sms, push |
| `notification_date` | TIMESTAMP | When to send the notification |

### Display & Metadata

| Field | Type | Description |
|-------|------|-------------|
| `name` | VARCHAR(255) | Task name/title |
| `task_number` | INT | **Display ID** - Auto-incrementing human-readable task number |
| `task_type` | VARCHAR(255) | String type identifier (legacy, e.g., "schedule", "mowing") |
| `type` | VARCHAR(100) | Additional type classification |
| `method` | VARCHAR | Execution method (e.g., mowing pattern) |
| `note` | VARCHAR(255) | User notes/comments |
| `reason` | VARCHAR(120) | Cancellation or modification reason |
| `timeout` | INT | Timeout value (purpose unclear from code) |

### Task Parameters

| Field | Type | Description |
|-------|------|-------------|
| `task_parameters` | MEDIUMTEXT | **JSON blob** containing task-specific parameters (mowing direction, height settings, etc.) |

### Computed Fields (Not Stored)

| Field | Description |
|-------|-------------|
| `sortingKey` | **Computed at runtime** - 3-character string based on: `task_start` (1/0) + `task_end` (1/0) + `running` (1/0). Used for sorting tasks in UI (completed tasks sort to bottom) |

### Versioning

| Field | Type | Description |
|-------|------|-------------|
| `version` | INT | Optimistic locking counter |
| `createdon` | TIMESTAMP | Created timestamp |
| `modifiedon` | TIMESTAMP | Last modified timestamp |

---

## Spraying Table (40 columns)

### Core Identifiers

| Field | Type | Description |
|-------|------|-------------|
| `idspraying` | BINARY(16) | Primary key (UUID) |
| `idgroup` | BINARY(16) | FK to group/course |
| `idsite` | BINARY(16) | FK to site |
| `idproduct` | BINARY(16) | FK to product being applied |
| `idstaff` | BINARY(16) | FK to operator |
| `idmachinery` | BINARY(16) | FK to sprayer/equipment |

### Quantity & Application

| Field | Type | Description |
|-------|------|-------------|
| `quantity` | DOUBLE | Total quantity of product used (in product units) |
| `quantity_area_unit` | DOUBLE(16,5) | **Quantity per area unit** - Application rate in kg/ha or L/ha |
| `speed` | DOUBLE(8,2) | **Application rate** - Same as quantity_area_unit in spraying routines. Maps to `application_rate` in routine templates |
| `dilution_volume` | | Dilution volume for the spray mix |

### Equipment Settings (Advanced Mode)

| Field | Type | Description |
|-------|------|-------------|
| `pressure` | DOUBLE(8,2) | **Spray pressure in bar** - Operating pressure of the sprayer |
| `truck_speed` | DOUBLE(8,2) | **Tractor/truck speed in km/h** - Ground speed during application |
| `truck_gear` | VARCHAR(255) | **Gear setting** - Transmission gear used (e.g., "L2", "H1") |
| `rpm` | DOUBLE | **Engine RPM** - PTO or engine speed for consistent output |

### Nozzle Configuration (Precision Mode)

| Field | Type | Description |
|-------|------|-------------|
| `nozzle_type` | INT | Legacy nozzle type identifier (integer) |
| `idnozzle_type` | INT | FK to `nozzle_type` lookup table |
| `idspacing` | INT | FK to `spacing` lookup table - Nozzle spacing configuration |
| `recommended_flow_rate` | DOUBLE | **Calculated** - Recommended nozzle flow rate based on speed and pressure |

### Tank & Dissolution Calculations

| Field | Type | Description |
|-------|------|-------------|
| `dissolution_ha` | DOUBLE | **Dissolution volume per hectare** (L/ha) - Water volume for spray mix |
| `total_dissolution` | DOUBLE | **Total dissolution volume** (L) - Total water needed for entire area |
| `product_amount_per_tank` | DOUBLE | **Product per tank** - Amount of product to add per tank fill |
| `number_of_tanks` | DOUBLE | **Tank count** - Number of tank fills required to cover the area |

### Nitrogen Tracking

| Field | Type | Description |
|-------|------|-------------|
| `n_units` | DOUBLE | **Nitrogen units applied** - Calculated from product N content × quantity |

### Location

| Field | Type | Description |
|-------|------|-------------|
| `hole` | VARCHAR(255) | **Hole/area identifiers** - Comma-separated list or JSON of target holes |

### Time & Status

| Field | Type | Description |
|-------|------|-------------|
| `doneon` | DATETIME | Date/time when spraying was completed |
| `task_start` | TIMESTAMP | Actual start time |
| `task_end` | TIMESTAMP | Actual end time |
| `estimate_time` | INT UNSIGNED | Estimated duration in minutes |
| `running` | TINYINT | 1 = completed, 0 = not completed |
| `status` | TINYINT | Status code (default 1) |
| `is_am` | TINYINT | Morning slot flag |
| `is_pm` | TINYINT | Afternoon slot flag |
| `is_over_time` | TINYINT | Overtime flag |

### Display & Metadata

| Field | Type | Description |
|-------|------|-------------|
| `name` | VARCHAR | Spraying name/description |
| `spraying_no` | INT | **Display ID** - Auto-incrementing per group. Used to identify spraying operations in UI (e.g., "Spraying #47") |
| `comment` | VARCHAR(255) | User comments/notes |
| `reason` | VARCHAR(100) | Cancellation reason |

### Routine Reference

| Field | Type | Description |
|-------|------|-------------|
| `spraying_routine_id` | BINARY(16) | FK to `spraying_routine` - Links to the routine template that generated this spraying (if applicable) |

---

## TOIL Table

| Field | Type | Description |
|-------|------|-------------|
| `idtoil` | BINARY(16) | Primary key |
| `iduser` | BINARY(16) | FK to user |
| `idtask` | BINARY(16) | FK to task (if TOIL from a task) |
| `idspraying` | BINARY(16) | FK to spraying (if TOIL from spraying) |
| `type` | VARCHAR | Type: "toil" or "overtime" |
| `hour` | INT | Hours component |
| `min` | INT | Minutes component |
| `starting_date_time` | DATETIME | When the TOIL/overtime occurred |

---

## Key Insights

### `speed` vs `truck_speed`
- `speed` = **Application rate** (L/ha or kg/ha) - how much product per hectare
- `truck_speed` = **Tractor ground speed** (km/h) - how fast the vehicle moves

### `spraying_no` Auto-Increment Logic
Generated per group: `MAX(spraying_no WHERE idgroup = ?) + 1`. Groups sprayings with same tags under same number.

### `is_all` Field Purpose
When `is_all = 1`, the task applies to ALL areas of the selected site, rather than specific hole/zone selections. Used in GrassRoot and general scheduling.

### `sortingKey` Computation
```php
$sortingKey = ($task_start ? '1' : '0') 
            . ($task_end ? '1' : '0') 
            . ($running ? '1' : '0');
// Results in: "000" (not started), "100" (in progress), "111" (completed)
```
