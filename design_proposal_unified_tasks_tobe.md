# Unified Task Architecture: Technical Implementation Specification

> **Document Version:** v5 (Implementation Ready)  
> **Status:** Approved for Development  
> **Last Updated:** 2025-12-22

---

## 1. Executive Summary

This specification defines the technical roadmap for implementing the **Unified Task Architecture** in Maya. It translates the architectural vision into concrete Implementation Steps, API Contracts, and Code Patterns.

**Primary Goal:** Replace the fragmented `task`/`spraying` tables with a normalized **Core-Extension** model, reducing API calls by 95% and enforcing strict compliance data integrity.

---

## 2. Database Schema Implementation

> **Note:** The Full SQL Schema is defined in [Appendix A](#appendix-a-full-sql-schema).

### 2.1 Core Concepts
*   **Kernel:** `tasks` table (UUID v7, Optimistic Locking).
*   **Extensions:** `task_ext_nutritional`, `task_ext_leave`.
*   **Graph:** `task_assignments`, `task_resources` (Junctions).

---

## 3. Backend Service Architecture (Laravel)

We will use the **Strategy Pattern** to handle domain-specific logic for different task types while keeping the controller lean.

### 3.1 Class Structure

```mermaid
classDiagram
    class TaskController {
        +index(Request)
        +store(CreateTaskRequest)
        +update(UpdateTaskRequest)
    }

    class TaskService {
        +create(DTO): Task
        +assignResources(Task, array)
    }

    interface TaskExtensionHandler {
        +validate(array data)
        +create(Task, array data)
        +update(Task, array data)
    }

    class NutritionalExtensionHandler {
        +create()
        +calculateNitrogen(data)
    }

    class LeaveExtensionHandler {
        +create()
        +handleApproval()
    }

    TaskController --> TaskService
    TaskService --> TaskExtensionHandler : delegates to
    TaskExtensionHandler <|-- NutritionalExtensionHandler
    TaskExtensionHandler <|-- LeaveExtensionHandler
```

### 3.2 Service Logic (`TaskService.php`)

```php
public function create(array $data): Task
{
    DB::beginTransaction();
    try {
        // 1. Create Core Task
        $task = Task::create($this->extractCoreData($data));

        // 2. Handle Assignments (Junction)
        if (!empty($data['assignments'])) {
            $this->syncAssignments($task, $data['assignments']);
        }

        // 3. Delegate to Extension Handler
        if ($handler = $this->getHandler($task->type_id)) {
            $handler->create($task, $data['extension_data'] ?? []);
        }

        DB::commit();
        return $task->load('assignments', 'extension');
    } catch (Exception $e) {
        DB::rollBack();
        throw $e;
    }
}
```

---

## 4. API Spec (JSON Contracts)

### 4.1 Create Unified Task (`POST /api/v2/tasks`)

**Request Payload:**

```json
{
  "title": "Spraying Hole 1-9",
  "type_id": 2, // SPRAYING
  "planned_start_at": "2025-12-23T08:00:00Z",
  "planned_end_at": "2025-12-23T12:00:00Z",
  "time_slot": "am",
  
  // Junction: Assignments (Replaces merge_key)
  "assignments": [
    { "user_id": "uuid-1", "role": "lead", "is_overtime": false },
    { "user_id": "uuid-2", "role": "support", "is_overtime": true }
  ],

  // Junction: Resources (Replaces idmachinery)
  "resources": [
    { "resource_id": "tractor-uuid", "operators": ["uuid-1"] },
    { "resource_id": "sprayer-uuid", "operators": ["uuid-1"] }
  ],

  // Extension Data (Polymorphic based on type_id)
  "extension_data": {
    "product_id": "chem-uuid",
    "quantity_applied": 50.0,
    "dosage_rate": 2.5,
    "weather_snapshot": {
      "air_temp_c": 18.5,
      "wind_speed_kmh": 12.0
    }
  }
}
```

### 4.2 List Tasks (`GET /api/v2/tasks`)

**Query Parameters:**
*   `start_date`, `end_date`: Range filter
*   `include`: `assignments,resources,extension` (Eager Loading)
*   `type_id`: Filter by type

**Response:**

```json
{
  "data": [
    {
      "id": "task-uuid",
      "title": "Spraying Hole 1-9",
      "assignments": [
        { "id": "user-1", "name": "John Doe", "pivot": { "role": "lead" } }
      ],
      "extension": {
        "nitrogen_units": 12.5,
        "weather_snapshot": { ... }
      }
    }
  ]
}
```

---

## 5. Frontend Migration Strategy (Vue.js)

### 5.1 Component Refactoring Map

| Component | Current State | Required Changes |
| :--- | :--- | :--- |
| **`ScheduleIndex.vue`** | Calls `/createSchedule` (Core 2) & `/createSpraying` (Core 1) | **Refactor**: Call single `POST /tasks` endpoint. Handle polymorphic form data. |
| **`TaskForm.vue`** | `merge_key` logic for multi-staff | **Simplify**: Send `assignments` array. Remove `merge_key` logic. |
| **`SprayingForm.vue`** | Separate View | **Merge**: Become a tab/section inside generic `TaskForm.vue` when Type = Spraying. |
| **`CalendarStore.js`** | Merges 3 API responses manually | **Cleanup**: specific `fetchTasks` action with standardized `Task` type. |

### 5.2 State Management (Pinia/Vuex)

*   **New Store**: `useTaskStore`
*   **Actions**: `fetchTasks(range)`, `createTask(payload)`, `patchTask(id, payload)`
*   **Getters**: `getTaskById`, `getTasksByResource(resourceId)`

---

## 6. Data Migration & Rollout Plan

### 6.1 Phase 1: Dual-Write (Zero Downtime)

Update the **Legacy Controllers** (`ScheduleController`, `SprayingController`) to write to *both* old and new tables.

```php
// Old Controller
public function createSchedule($request) {
    $oldTask = $this->repo->create($request); // Existing Logic
    
    // NEW: Async Job to sync to new tables
    SyncTaskToUnifiedTable::dispatch($oldTask);
    
    return $response;
}
```

### 6.2 Phase 2: Bulk Migration Script

**Script Logic:** `migrate:unified_tasks`
1.  **Chunk 1**: Load `task` grouped by `merge_key`.
    *   Create `tasks` row.
    *   Iterate group members -> Create `task_assignments`.
2.  **Chunk 2**: Load `spraying` records.
    *   Match to `tasks` via heuristic (time + site) OR direct migration.
    *   Populate `task_ext_nutritional`.
3.  **Verification**: Compare `count(old_tasks)` vs `count(new_tasks)`.

### 6.3 Phase 3: Switch Reads (Validation)

*   Update **Summary Tables** (Reporting) to read from `tasks`.
*   Compare reports generated from Old vs New tables.
*   Once parity is confirmed (within 0.1% cost variance), proceed.

---

## 7. Security & Permissions

| Action | Permission Required | Scope |
| :--- | :--- | :--- |
| `tasks.view.any` | `view_schedule` | Can read core task data |
| `tasks.view.financial` | `view_financials` | Can read `total_resource_cost` |
| `tasks.create.spraying`| `create_spraying` | Can create tasks with type=SPRAYING |
| `tasks.approve.leave` | `approve_leave` | Can update `task_ext_leave.approval_status` |

---

## 8. MVP Scope (v1) vs Future Migration (v2)

### v1 MVP Scope — Unified Tasks

The following entities are **included** in the v1 unified task architecture:

| Entity | v1 Status | Notes |
|--------|-----------|-------|
| `tasks` | ✅ Unified | Core table replaces legacy `task` |
| `task_assignments` | ✅ Unified | Replaces `merge_key` cloning |
| `task_resources` | ✅ Unified | Machine assignments |
| `task_resource_operators` | ✅ Unified | Operator-machine compliance |
| `task_locations` | ✅ Unified | Multi-site/hole support |
| `task_ext_nutritional` | ✅ Unified | Replaces `spraying` table |
| `task_ext_leave` | ✅ Unified | Leave management |

### v2 Migration Scope — Deferred Systems

The following remain as **separate tables** in v1 for stability:

| Entity | v1 Status | v2 Plan |
|--------|-----------|---------|
| `toil` | 🔄 Separate | Migrate to `task_assignments` or dedicated extension |
| `daily_routine` | 🔄 Separate | Create `task_templates` for recurring task generation |
| `daily_routine_tasks` | 🔄 Separate | Part of template system |
| `daily_routine_site/hole/area` | 🔄 Separate | Part of template system |
| `daily_routine_staff/machine` | 🔄 Separate | Part of template system |
| `spraying_routine` | 🔄 Separate | Integrate with `task_ext_nutritional.routine_id` |
| `spraying_routine_*` | 🔄 Separate | Part of spraying template system |

### v2 Migration Strategy

```
v1 (MVP):
┌─────────────────────┐     ┌──────────────────────┐
│   Unified Tasks     │     │  Separate Systems    │
│   ─────────────     │     │  ────────────────    │
│   tasks             │     │  toil                │
│   task_assignments  │     │  daily_routine*      │
│   task_ext_*        │     │  spraying_routine*   │
└─────────────────────┘     └──────────────────────┘
         │                            │
         │  routine_id FK ───────────►│
         │  (prepared for v2)         │
         ▼                            ▼
v2 (Full Migration):
┌──────────────────────────────────────────────────┐
│              Unified Task System                 │
│   ──────────────────────────────────────         │
│   tasks + task_templates (from routines)         │
│   task_assignments + embedded TOIL               │
│   task_ext_nutritional (full routine support)    │
└──────────────────────────────────────────────────┘
```

### Compatibility Notes

1. **TOIL in v1:** The separate `toil` table continues to work. The `routine_id` field in `task_ext_nutritional` provides a forward-compatible FK.

2. **Routines in v1:** Daily/Spraying routines continue generating tasks via existing logic. The `routine_id` allows future traceability.

3. **Phyto License Validation:** In v1, spraying task creation validates `user.phyto_license` at service level (not FK constraint).

---

## Appendix A: Full SQL Schema

### Core Table: `tasks`
```sql
CREATE TABLE `tasks` (
    `id` BINARY(16) NOT NULL PRIMARY KEY,
    `tenant_id` BINARY(16) NOT NULL,
    `group_id` BINARY(16) NOT NULL,                 -- idgroup: FK to course/group
    `site_id` BINARY(16) NULL,                      -- Primary site (optional if multi-site via task_locations)
    `action_id` BINARY(16) NOT NULL,                -- FK to action table (mowing, aeration, etc.)
    `parent_task_id` BINARY(16) NULL,               -- For sub-task hierarchies
    
    -- Type and Status
    `type_id` SMALLINT UNSIGNED NOT NULL,           -- 1=regular, 2=spraying, 3=leave
    `status_id` SMALLINT UNSIGNED NOT NULL DEFAULT 1, -- 1=todo, 2=in_progress, 3=completed, 4=not_completed
    `running` BOOLEAN DEFAULT FALSE,                -- Legacy: 1=completed (for mobile compatibility)
    
    -- Display Fields
    `title` VARCHAR(255) NOT NULL,
    `task_number` INT UNSIGNED NULL,                -- Auto-increment display ID per group (e.g., "Task #47")
    `is_all_areas_selected` BOOLEAN DEFAULT FALSE,  -- True = applies to ALL areas of site, not specific selections
    
    -- Time Scheduling
    `time_slot` ENUM('am', 'pm', 'all_day') DEFAULT 'all_day',
    `planned_start_at` DATETIME NOT NULL,
    `planned_end_at` DATETIME NOT NULL,
    `actual_start_at` DATETIME NULL,                -- Set when task begins on mobile
    `actual_end_at` DATETIME NULL,                  -- Set when task completes on mobile
    `estimated_duration_minutes` INT UNSIGNED NULL, -- estimate_time in minutes
    
    -- Notifications
    `notify_at` DATETIME NULL,                      -- When to send notification
    `notification_channel` ENUM('email', 'sms', 'push', 'none') DEFAULT 'email',
    
    -- Metadata
    `notes` TEXT NULL,                              -- User notes/comments
    `cancellation_reason` TEXT NULL,                -- Reason for cancellation/modification
    `parameters` JSON NULL,                         -- Task-specific params (mowing direction, height, etc.)
    `method` VARCHAR(100) NULL,                     -- Execution method (e.g., mowing pattern)
    
    -- Recurrence (for recurring tasks)
    `recurrence_rule` TEXT NULL,                    -- iCal RRULE (e.g., "FREQ=WEEKLY;BYDAY=MO,WE")
    `next_execution_at` DATETIME NULL,              -- Next scheduled occurrence
    `last_execution_at` DATETIME NULL,              -- Most recent execution
    
    -- Versioning & Audit
    `version` INT UNSIGNED NOT NULL DEFAULT 0,      -- Optimistic locking counter
    `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    `updated_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- Indexes
    INDEX `idx_tenant_range` (`tenant_id`, `planned_start_at`),
    INDEX `idx_action` (`action_id`),
    INDEX `idx_parent` (`parent_task_id`),
    INDEX `idx_status` (`status_id`),
    INDEX `idx_group` (`group_id`),
    
    -- Foreign Keys
    CONSTRAINT `fk_task_parent` FOREIGN KEY (`parent_task_id`) REFERENCES `tasks` (`id`) ON DELETE SET NULL
) ENGINE=InnoDB;

-- NOTE: sortingKey is COMPUTED at runtime, not stored:
-- sortingKey = (actual_start_at ? '1' : '0') + (actual_end_at ? '1' : '0') + (running ? '1' : '0')
-- Used for UI sorting: "000"=not started, "100"=in progress, "111"=completed
```

### Assignments: `task_assignments`
```sql
CREATE TABLE `task_assignments` (
    `task_id` BINARY(16) NOT NULL,
    `user_id` BINARY(16) NOT NULL,
    `role` ENUM('lead', 'support') DEFAULT 'lead',
    
    -- Time Tracking (FRS 3.3)
    `is_overtime` BOOLEAN DEFAULT FALSE,
    `hours_worked` DECIMAL(8,2) NULL,
    `hours_overtime` DECIMAL(8,2) NULL,             -- Overtime hours for this assignment
    
    -- TOIL Tracking (FRS 3.3)
    `toil_hours` DECIMAL(8,2) NULL,                 -- TOIL hours accrued/used
    `toil_requested` BOOLEAN DEFAULT FALSE,         -- Whether TOIL was requested for this work
    
    PRIMARY KEY (`task_id`, `user_id`),
    CONSTRAINT `fk_assign_task` FOREIGN KEY (`task_id`) REFERENCES `tasks` (`id`) ON DELETE CASCADE
) ENGINE=InnoDB;
```

### Extension: `task_ext_nutritional`
```sql
CREATE TABLE `task_ext_nutritional` (
    `task_id` BINARY(16) NOT NULL PRIMARY KEY,
    `product_id` BINARY(16) NOT NULL,
    
    -- Quantity and Dosage
    `quantity_applied` DECIMAL(12, 4) NOT NULL,     -- Total product quantity used
    `quantity_unit` VARCHAR(20) DEFAULT 'kg/ha',    -- Unit: kg/ha, L/ha, g/sqm
    `application_rate` DECIMAL(10, 4) NULL,         -- 'speed' field: Rate of application (L/ha or kg/ha)
    `quantity_area_unit` DECIMAL(16, 5) NULL,       -- Quantity per area unit (same as application_rate)
    
    -- Operational Mode
    `operational_mode` ENUM('standard', 'advanced', 'precision') DEFAULT 'standard',
    
    -- Advanced Spraying Mode Fields
    `flow_rate_lpm` DECIMAL(8, 2) NULL,             -- Nozzle flow rate in Litres Per Minute
    `tractor_speed_kmh` DECIMAL(8, 2) NULL,         -- 'truck_speed': Ground speed of tractor in km/h
    `rpm` DOUBLE NULL,                              -- Engine/PTO RPM for consistent output
    `gear` VARCHAR(255) NULL,                       -- 'truck_gear': Transmission gear (e.g., "L2", "H1")
    `pressure_bar` DECIMAL(8, 2) NULL,              -- Spray pressure in bar
    
    -- Dissolution/Tank Calculations
    `dissolution_volume_per_ha` DOUBLE NULL,        -- 'dissolution_ha': Water volume per hectare (L/ha)
    `total_dissolution_volume` DOUBLE NULL,         -- 'total_dissolution': Total water needed (L)
    `tank_size_litres` DECIMAL(10, 2) NULL,         -- Tank capacity in litres
    `product_amount_per_tank` DOUBLE NULL,          -- Product to add per tank fill
    `number_of_tanks` DOUBLE NULL,                  -- Tank fills required (can be fractional)
    `recommended_flow_rate` DOUBLE NULL,            -- Calculated: optimal nozzle flow rate
    
    -- Precision Mode Fields (Nozzle Configuration)
    `nozzle_type_id` INT NULL,                      -- FK to nozzle_type lookup table (INT, not UUID)
    `nozzle_type_legacy` INT NULL,                  -- Legacy 'nozzle_type' integer field
    `spacing_id` INT NULL,                          -- FK to spacing lookup table (nozzle spacing config)
    `nozzle_spacing_cm` DECIMAL(6, 2) NULL,         -- Nozzle spacing in cm (if not using lookup)
    
    -- Nitrogen Tracking
    `nitrogen_units` DOUBLE NULL,                   -- 'n_units': Calculated N from product N content × quantity
    
    -- Weather Snapshot (14 metrics captured at spray time)
    `weather_snapshot` JSON NULL,                   -- {air_temp, wind_speed, humidity, etc.}
    
    -- Location (legacy field)
    `hole_ids` VARCHAR(255) NULL,                   -- 'hole': Comma-separated or JSON of target hole IDs
    
    -- Display & Reference
    `spraying_number` INT UNSIGNED NULL,            -- 'spraying_no': Auto-increment per group (e.g., "Spraying #47")
    `completed_at` DATETIME NULL,                   -- 'doneon': When spraying was completed
    `comments` VARCHAR(255) NULL,                   -- User comments
    
    -- Stock Consumption Reference
    `stock_transaction_id` BINARY(16) NULL,         -- FK to stock ledger for inventory decrement
    
    -- Routine Reference (v2 migration)
    `routine_id` BINARY(16) NULL,                   -- 'spraying_routine_id': FK to routine template
    
    -- Indexes
    INDEX `idx_product` (`product_id`),
    INDEX `idx_routine` (`routine_id`),
    INDEX `idx_spraying_number` (`spraying_number`),
    
    CONSTRAINT `fk_ext_nutr` FOREIGN KEY (`task_id`) REFERENCES `tasks` (`id`) ON DELETE CASCADE
) ENGINE=InnoDB;

-- NOTE: 'speed' vs 'truck_speed' distinction from AS-IS:
-- speed = application_rate (L/ha or kg/ha) - how much product per hectare
-- truck_speed = tractor_speed_kmh (km/h) - ground speed of vehicle
```

### Extension: `task_ext_leave` (FRS 3.3)
```sql
CREATE TABLE `task_ext_leave` (
    `task_id` BINARY(16) NOT NULL PRIMARY KEY,
    `leave_type_id` BINARY(16) NOT NULL,
    `leave_type_name` VARCHAR(100) NOT NULL,
    `approval_status` ENUM('pending', 'approved', 'rejected') DEFAULT 'pending',
    `approved_by` BINARY(16) NULL,
    `approved_at` DATETIME NULL,
    `leave_days` DECIMAL(4, 2) NULL,                -- Supports half-days
    `leave_hours` DECIMAL(6, 2) NULL,
    `comments` TEXT NULL,
    CONSTRAINT `fk_ext_leave` FOREIGN KEY (`task_id`) REFERENCES `tasks` (`id`) ON DELETE CASCADE
) ENGINE=InnoDB;
```

### Junction: `task_locations` (FRS 3.1 Multi-Location)
```sql
CREATE TABLE `task_locations` (
    `task_id` BINARY(16) NOT NULL,
    `site_id` BINARY(16) NOT NULL,
    `hole_id` BINARY(16) NULL,                      -- NULL for site-level tasks
    `area_id` BINARY(16) NULL,                      -- Zone/Area within hole
    `surface_area_sqm` DECIMAL(12, 2) NULL,         -- For pro-rata calculations
    `hours_allocated` DECIMAL(6, 2) NULL,           -- Distributed hours per location
    PRIMARY KEY (`task_id`, `site_id`, COALESCE(`hole_id`, ''), COALESCE(`area_id`, '')),
    CONSTRAINT `fk_loc_task` FOREIGN KEY (`task_id`) REFERENCES `tasks` (`id`) ON DELETE CASCADE
) ENGINE=InnoDB;
```

### Junction: `task_resource_operators` (FRS 3.4 Fleet Compliance)
```sql
CREATE TABLE `task_resource_operators` (
    `id` BINARY(16) NOT NULL PRIMARY KEY,
    `task_id` BINARY(16) NOT NULL,
    `resource_id` BINARY(16) NOT NULL,              -- Machine/Equipment
    `operator_id` BINARY(16) NOT NULL,              -- Staff member
    `certification_ref` VARCHAR(100) NULL,          -- Operator certification for compliance
    `hours_operated` DECIMAL(6, 2) NULL,
    CONSTRAINT `fk_resop_task` FOREIGN KEY (`task_id`) REFERENCES `tasks` (`id`) ON DELETE CASCADE,
    UNIQUE KEY `uk_task_resource_operator` (`task_id`, `resource_id`, `operator_id`)
) ENGINE=InnoDB;
```

---

## Appendix B: FRS Traceability Matrix

This section maps Functional Requirements (from `functional_requirements_specification.md`) to database schema elements.

### Task Management (FRS 3.1)

| Requirement | Schema Element | Coverage |
|-------------|---------------|----------|
| Task status flow | `tasks.status_id` → lookup table | ✅ |
| Task types (site/tenant-based) | Derived from `task_locations` presence | ✅ |
| Multi-location tasks | `task_locations` junction table | ✅ |
| Planned duration | `tasks.planned_start_at`, `planned_end_at` | ✅ |
| Actual duration | `tasks.actual_start_at`, `actual_end_at` | ✅ |
| Labour calculation | `task_assignments.hours_worked` × staff count | ✅ |

### Nutritional Inputs (FRS 3.2)

| Requirement | Schema Element | Coverage |
|-------------|---------------|----------|
| Standard mode | `task_ext_nutritional.operational_mode = 'standard'` | ✅ |
| Advanced mode fields | `flow_rate_lpm`, `tractor_speed_kmh`, `rpm`, `tank_size_litres` | ✅ |
| Precision mode fields | `nozzle_type`, `nozzle_spacing_cm`, `nozzle_flow_rate` | ✅ |
| Nitrogen tracking | `task_ext_nutritional.nitrogen_units` | ✅ |
| Stock consumption | `task_ext_nutritional.stock_transaction_id` | ✅ |
| Weather capture | `task_ext_nutritional.weather_snapshot` (14 metrics JSON) | ✅ |

### Labour & Time Tracking (FRS 3.3)

| Requirement | Schema Element | Coverage |
|-------------|---------------|----------|
| Time slots | `tasks.time_slot` ENUM('am', 'pm', 'all_day') | ✅ |
| Overtime flag | `task_assignments.is_overtime` | ✅ |
| Overtime hours | `task_assignments.hours_overtime` | ✅ |
| TOIL tracking | `task_assignments.toil_hours`, `toil_requested` | ✅ |
| Hours worked | `task_assignments.hours_worked` | ✅ |
| Leave management | `task_ext_leave` extension table | ✅ |
| Leave approval | `task_ext_leave.approval_status`, `approved_by` | ✅ |

### Fleet Management (FRS 3.4)

| Requirement | Schema Element | Coverage |
|-------------|---------------|----------|
| Resource assignment | `task_resources` junction table | ✅ |
| Operator-machine link | `task_resource_operators` junction table | ✅ |
| Certification tracking | `task_resource_operators.certification_ref` | ✅ |
| Hours operated | `task_resource_operators.hours_operated` | ✅ |

### Weather & Environmental (FRS 3.5)

| Requirement | Schema Element | Coverage |
|-------------|---------------|----------|
| 14 weather metrics | `task_ext_nutritional.weather_snapshot` JSON | ✅ |

### Reporting (FRS 3.6)

| Requirement | Schema Element | Coverage |
|-------------|---------------|----------|
| Summary table compatibility | All fields map to `summary_tasks` structure | ✅ |
| Record types | `tasks.type_id` distinguishes task_schedule/task_spraying | ✅ |
| Work hours aggregation | `task_assignments.hours_worked` | ✅ |
| Cost tracking | Calculated from resource + labour rates | ✅ |

### Business Rules (FRS Section 6)

| Rule ID | Rule | Implementation |
|---------|------|----------------|
| BR-001 | Planned Labour = Duration × Staff | Query: `SUM(hours_worked)` per task | 
| BR-002 | Hours per Location = Duration / Locations | `task_locations.hours_allocated` |
| BR-003 | Surface normalisation (÷10000) | `task_locations.surface_area_sqm / 10000` |
| BR-004 | Overtime determination | `task_assignments.is_overtime` OR exceeds schedule |
| BR-005 | Stock consumption by type | `task_ext_nutritional.stock_transaction_id` |
| BR-006 | Pro-rata quantity | `task_locations.surface_area_sqm` for distribution |
