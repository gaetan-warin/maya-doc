# Field Mapping & Migration SQL Reference
## Legacy (Model 2) to Target (Model 3)

| Document Details | |
| :--- | :--- |
| **Project** | Maya Core 2.0 |
| **Module** | Scheduling & Task Management |
| **Scope** | Database Schema Migration |
| **Status** | Authoritative Reference |

---

## 1. Overview
This document maps Legacy tables to the Target Model 3 schema and provides the **SQL Statements** required to execute the data migration.

**Migration Logic:**
1.  **Parents First:** We migrate Parent Tasks first to generate new UUIDs.
2.  **legacy_task_id:** We store the old UUID to allow joining child rows (staff/machines) later.
3.  **UUID Generation:** The SQL examples use `UUID()`. In production, the PHP script should substitute this with `Str::orderedUuid()`.

---

## 2. General Tasks Migration (`task` → `tasks`)

We first migrate the "Parent" rows from the legacy `task` table. A row is a parent if `idparent_task` is NULL.

### 2.1 Core Field Mapping

| Legacy `task` | Target `tasks` | Explanation |
| :--- | :--- | :--- |
| `idtask` | `legacy_task_id` | **Audit:** We keep the old ID to link child rows later. |
| `merge_key` | `legacy_merge_key` | **Audit:** Kept for safety and backfill validation. |
| `idgroup` | `idgroup` | **Copy:** No change. Defines the partition. |
| `idaction` | `action_id` | **Copy:** No change. |
| `task_start` | `actual_start` | **Map:** `task_start` represents the actual punch-in time. |
| `task_end` | `actual_end` | **Map:** `task_end` represents actual completion. |
| `estimate_time` | `planned_minutes` | **Rename:** Clearer naming. |
| `note` | `notes` | **Copy:** No change. |
| `running` | `status` | **Transform:** `0` -> `'PLANNED'`, `1` -> `'RUNNING'`, if `task_end` not null -> `'COMPLETED'`. |
| `createdon` | `created_at` | **Copy:** No change. |

### 2.2 SQL Migration Statement

```sql
INSERT INTO tasks (
    id, 
    idgroup, 
    type_id, 
    action_id, 
    title,
    planned_minutes, 
    actual_start, 
    actual_end, 
    status, 
    notes, 
    legacy_task_id, 
    legacy_merge_key, 
    version, 
    created_at
)
SELECT 
    UUID(),                         -- Generate NEW Task ID
    t.idgroup, 
    'GENERAL',                      -- Constant type
    t.idaction, 
    'Legacy Task',                  -- Default title (optional: join action name)
    t.estimate_time, 
    t.task_start, 
    t.task_end, 
    CASE 
        WHEN t.task_end IS NOT NULL THEN 'COMPLETED'
        WHEN t.running = 1 THEN 'RUNNING'
        ELSE 'PLANNED' 
    END AS status,
    t.note, 
    t.idtask,                       -- Store Legacy ID for Joins
    t.merge_key, 
    1,                              -- Initial Version
    t.createdon
FROM task t
WHERE t.idparent_task IS NULL;      -- Only migrate Parent Rows
```

---

## 3. Site Locations (`task` → `task_locations`)

Legacy tasks have an `idsite`. We move this to the `task_locations` junction table.

### 3.1 Mapping
| Legacy `task` | Target `task_locations` | Explanation |
| :--- | :--- | :--- |
| `idtask` (Parent) | `task_id` | **Link:** Map via `legacy_task_id`. |
| `idsite` | `site_id` | **Copy:** The location of the task. |

### 3.2 SQL Statement

```sql
INSERT INTO task_locations (id, task_id, site_id)
SELECT 
    UUID(), 
    new_t.id, 
    old_t.idsite
FROM task old_t
JOIN tasks new_t ON new_t.legacy_task_id = old_t.idtask  -- Join to get new ID
WHERE old_t.idparent_task IS NULL 
AND old_t.idsite IS NOT NULL;
```

---

## 4. Staff Assignments (`task` → `task_assignments`)

In Legacy, staff are separate rows with `idparent_task` pointing to the main task.

### 4.1 Mapping
| Legacy `task` (Child) | Target `task_assignments` | Explanation |
| :--- | :--- | :--- |
| `idparent_task` | `task_id` | **Link:** Follow the parent pointer to find the new Task ID. |
| `idstaff` | `user_id` | **Copy:** The assigned user. |
| `is_over_time` | `is_over_time` | **Copy:** No change. |

### 4.2 SQL Statement

```sql
INSERT INTO task_assignments (id, task_id, user_id, role, is_over_time)
SELECT 
    UUID(), 
    parent_t.id,            -- The NEW ID of the parent task
    child_t.idstaff, 
    'WORKER',               -- Default Role
    IFNULL(child_t.is_over_time, 0)
FROM task child_t
JOIN tasks parent_t ON parent_t.legacy_task_id = child_t.idparent_task -- Link Child -> Old Parent -> New Parent
WHERE child_t.idstaff IS NOT NULL;
```

---

## 5. Machine Resources (`task` → `task_resources`)

Similar to staff, machines are child rows.

### 5.1 Mapping
| Legacy `task` (Child) | Target `task_resources` | Explanation |
| :--- | :--- | :--- |
| `idparent_task` | `task_id` | **Link:** Follow parent pointer. |
| `idmachinery` | `resource_id` | **Copy:** The assigned machine. |

### 5.2 SQL Statement

```sql
INSERT INTO task_resources (id, task_id, resource_id, resource_type)
SELECT 
    UUID(), 
    parent_t.id, 
    child_t.idmachinery, 
    'MACHINE'
FROM task child_t
JOIN tasks parent_t ON parent_t.legacy_task_id = child_t.idparent_task
WHERE child_t.idmachinery IS NOT NULL;
```

---

## 6. Spraying Records (`spraying` → `tasks` + `ext`)

Spraying is a separate table. We migrate it into `tasks` (concept) and `task_ext_nutritional` (details).

### 6.1 Core Task Mapping
| Legacy `spraying` | Target `tasks` | Explanation |
| :--- | :--- | :--- |
| `idspraying` | `legacy_spraying_id` | **Audit:** Store old ID. |
| `idsite` | `task_locations.site_id` | **Move:** To locations table. |
| `name` | `title` | **Copy:** Use spraying name as title. |
| `doneon` | `actual_start` | **Map:** Execution date. |

### 6.2 SQL Statement (Core)

```sql
INSERT INTO tasks (
    id, idgroup, type_id, title, actual_start, status, legacy_spraying_id, version, created_at
)
SELECT 
    UUID(), 
    s.idgroup,          -- NOTE: Ensure spraying has idgroup (from join if needed)
    'SPRAYING', 
    s.name, 
    s.doneon, 
    'COMPLETED', 
    s.idspraying, 
    1, 
    s.doneon
FROM spraying s;
```

### 6.3 Extension Mapping
| Legacy `spraying` | Target `task_ext_nutritional` | Explanation |
| :--- | :--- | :--- |
| `idspraying` | `task_id` | **Link:** New Task ID. |
| `nozzle_type` | `nozzle_type_id` | **Copy:** Direct map. |
| `quantity/dilution` | `parameters` | **JSON/Text:** encode these into the `parameters` or `compliance_data` column. |

### 6.4 SQL Statement (Extension)

```sql
INSERT INTO task_ext_nutritional (
    task_id, nozzle_type_id, speed, rpm, pressure, wind_speed, temperature
)
SELECT 
    t.id,               -- The New Task ID created in 6.2
    s.nozzle_type, 
    s.speed, 
    s.rpm, 
    s.pressure, 
    s.wind_speed, 
    s.temperature
FROM spraying s
JOIN tasks t ON t.legacy_spraying_id = s.idspraying;
```

---

## 7. Product Links (`product_task` → `task_products`)

Legacy links products to `merge_key`. We must resolve this to the new `task_id`.

### 7.1 Mapping
| Legacy `product_task` | Target `task_products` | Explanation |
| :--- | :--- | :--- |
| `merge_key_hash` | `task_id` | **Link:** Resolve via `tasks.legacy_merge_key`. |
| `idproduct` | `product_id` | **Copy:** No change. |
| `quantity` | `quantity` | **Copy:** No change. |

### 7.2 SQL Statement

```sql
INSERT INTO task_products (id, task_id, product_id, quantity)
SELECT
    UUID(),
    t.id,
    pt.idproduct,
    pt.quantity
FROM product_task pt
-- We need to link product_task to the new task.
-- PROBLEM: product_task uses a HASH of the merge_key, not the merge_key itself.
-- VALIDATION REQUIRED: Does legacy table store the raw merge_key?
-- IF NOT, the PHP script must re-hash new tasks to find the match.
-- ASSUMING 'merge_key' exists in product_task or we migrate via PHP logic:
JOIN tasks t ON t.legacy_merge_key = pt.merge_key;
```
*Note: If `product_task` only has a hash, this step **must** be done in PHP/Code, not SQL.*

---

## 8. Follow-up Records (`report_follow_up` → `tasks` + `task_ext_followup`)

Follow-ups are migrated from the separate `report_follow_up` table into the unified task system.

### 8.1 Core Task Mapping

| Legacy `report_follow_up` | Target `tasks` | Explanation |
| :--- | :--- | :--- |
| `idreport_follow_up` | `legacy_followup_id` (in ext table) | **Audit:** Store old ID for traceability. |
| `idreport` | `task_ext_followup.report_id` | **Link:** Reference to parent incident report. |
| `iduser` | `task_assignments.user_id` | **Move:** To assignments table. |
| `date` | `planned_start` | **Map:** Scheduled follow-up date. |
| `comment` | `notes` | **Copy:** Follow-up notes. |
| `total_duration` | `planned_minutes` | **Transform:** Convert TIME to minutes. |
| `total_cost` | `task_ext_followup.total_cost` | **Move:** To extension table. |
| `status` | `task_ext_followup.followup_status` | **Copy:** 'todo' or 'completed'. |
| `status` | `status` | **Transform:** 'todo' → 'PLANNED', 'completed' → 'COMPLETED'. |

### 8.2 SQL Statement (Core Task)

```sql
-- Step 1: Insert into unified tasks table
INSERT INTO tasks (
    id,
    idgroup,
    type_id,
    title,
    planned_start,
    planned_minutes,
    status,
    notes,
    version,
    created_at,
    updated_at
)
SELECT
    UUID(),                                           -- Generate NEW Task ID
    r.idgroup,                                        -- From report's group
    'FOLLOWUP',                                       -- Constant type
    CONCAT('Follow-up: ', COALESCE(i.name, 'Incident Report')),  -- Title from incident
    rf.date,                                          -- Scheduled date
    CASE
        WHEN rf.total_duration IS NOT NULL
        THEN TIME_TO_SEC(rf.total_duration) / 60
        ELSE NULL
    END,                                              -- Convert TIME to minutes
    CASE rf.status
        WHEN 'completed' THEN 'COMPLETED'
        ELSE 'PLANNED'
    END,                                              -- Map status
    rf.comment,                                       -- Notes
    1,                                                -- Initial version
    rf.created_at,
    rf.updated_at
FROM report_follow_up rf
JOIN report r ON rf.idreport = r.idreport
LEFT JOIN incident i ON r.idincident = i.idincident;
```

### 8.3 SQL Statement (Extension Table)

```sql
-- Step 2: Insert into extension table
-- Note: This requires a join to find the newly created task IDs
-- Best done in PHP where we can track the UUID mapping

INSERT INTO task_ext_followup (
    task_id,
    report_id,
    total_cost,
    followup_status,
    legacy_followup_id
)
SELECT
    t.id,                                             -- New Task ID (from mapping)
    rf.idreport,
    rf.total_cost,
    rf.status,
    rf.idreport_follow_up                             -- Legacy ID for audit
FROM report_follow_up rf
JOIN tasks t ON t.type_id = 'FOLLOWUP'
    AND t.notes = rf.comment
    AND DATE(t.planned_start) = rf.date
    AND t.created_at = rf.created_at;                 -- Match by multiple fields

-- RECOMMENDED: Use PHP migration script for accurate UUID mapping
```

### 8.4 SQL Statement (Staff Assignments)

```sql
-- Step 3: Migrate staff assignments
INSERT INTO task_assignments (id, task_id, user_id, role)
SELECT
    UUID(),
    tef.task_id,
    rf.iduser,
    'WORKER'                                          -- Default role
FROM report_follow_up rf
JOIN task_ext_followup tef ON tef.legacy_followup_id = rf.idreport_follow_up
WHERE rf.iduser IS NOT NULL;
```

### 8.5 SQL Statement (Locations)

```sql
-- Step 4: Migrate locations from parent report
INSERT INTO task_locations (id, task_id, site_id)
SELECT
    UUID(),
    tef.task_id,
    r.idsite
FROM task_ext_followup tef
JOIN report r ON r.idreport = tef.report_id
WHERE r.idsite IS NOT NULL;
```

### 8.6 SQL Statement (Images)

```sql
-- Step 5: Migrate follow-up images
INSERT INTO task_images (id, task_id, image_url, image_type, created_at)
SELECT
    UUID(),
    tef.task_id,
    rfi.image_url,
    'evidence',                                       -- Default type for follow-up images
    NOW()
FROM report_follow_up_image rfi
JOIN task_ext_followup tef ON tef.legacy_followup_id = rfi.idreport_follow_up;
```

### 8.7 Post-Migration: Update Report Status

```sql
-- Ensure report.follow_up_status matches the migrated follow-up status
UPDATE report r
JOIN task_ext_followup tef ON tef.report_id = r.idreport
JOIN tasks t ON t.id = tef.task_id
SET r.follow_up_status = CASE
    WHEN t.status = 'COMPLETED' THEN 3
    WHEN t.status = 'PLANNED' THEN 2
    ELSE r.follow_up_status
END;
```

---

## 9. Migration Order Summary

Execute migrations in this order to maintain referential integrity:

| Step | Source | Target | Notes |
| :--- | :--- | :--- | :--- |
| 1 | `task` (parents) | `tasks` | GENERAL type, parent rows only |
| 2 | `task` (sites) | `task_locations` | From parent task.idsite |
| 3 | `task` (staff children) | `task_assignments` | Child rows with idstaff |
| 4 | `task` (machine children) | `task_resources` | Child rows with idmachinery |
| 5 | `taggable` | `task_tags` | Link via legacy_task_id |
| 6 | `spraying` | `tasks` | SPRAYING type |
| 7 | `spraying` | `task_ext_nutritional` | Extension fields |
| 8 | `product_task` | `task_products` | Via merge_key or PHP |
| 9 | `report_follow_up` | `tasks` | FOLLOWUP type |
| 10 | `report_follow_up` | `task_ext_followup` | Extension fields |
| 11 | `report_follow_up` | `task_assignments` | Staff from follow-up |
| 12 | `report` | `task_locations` | Site from parent report |
| 13 | `report_follow_up_image` | `task_images` | Evidence images |
