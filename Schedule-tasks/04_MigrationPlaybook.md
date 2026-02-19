# Migration Playbook
## Maya — Transition from Model 2 to Model 3

| Document Details | |
| :--- | :--- |
| **Project** | Maya Core 2.0 |
| **Module** | Scheduling & Task Management |
| **Version** | 1.0 |
| **Status** | Execution Guide |
| **Target Audience** | Tech Leads, DevOps, Senior Developers |

---

## 1. Executive Summary

This document provides a **step-by-step execution plan** to migrate the Maya Scheduling module from the current legacy architecture (**Model 2**) to the target unified architecture (**Model 3**).

### Migration Objectives
1.  **Zero Downtime:** No service interruption during transition.
2.  **Data Integrity:** All historical data is preserved and mapped.
3.  **Gradual Rollout:** Feature flags enable progressive adoption.
4.  **Rollback Safety:** Ability to revert to Model 2 if issues arise.

### Migration Approach: **Dual-Write with Progressive Read Switch**

```
┌─────────────────────────────────────────────────────────────────────┐
│                        MIGRATION TIMELINE                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Phase 0      Phase 1      Phase 2      Phase 3      Phase 4       │
│  ────────     ────────     ────────     ────────     ────────      │
│  Preparation  Schema       Dual-Write   Backfill     Read Switch   │
│  (1-2 days)   Deploy       + Validation + Validation + Cleanup     │
│               (1 day)      (1-2 weeks)  (2-3 days)   (1 week)      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Phase 0: Preparation & Decision Making

**Duration:** 1-2 days
**Owner:** Tech Lead

### 2.1 Confirm Source of Truth Flows

| Question | Answer |
| :--- | :--- |
| Which UI entry points create Tasks? | `scheduleSaveTasks.js` → `POST /createSchedule` |
| Which UI entry points create Spraying? | Spraying page → `POST /createSprayingRecords` |
| Which backend endpoints write data? | `ScheduleController`, `SprayingController` |

### 2.2 Document the Merge Key Algorithm

**Action:** Extract and document the exact `merge_key` generation logic from `scheduleSaveTasks.js`.

```javascript
// Pseudo-code (from frontend)
mergeKey = concat(
  siteIds.map(removeHyphens).join(''),
  holeIds.map(removeHyphens).join(''),
  actionId.removeHyphens(),
  date.format('YYYYMMDD'),
  randomSalt()
)
```

### 2.3 Key Decisions

| Decision | Recommended Choice | Rationale |
| :--- | :--- | :--- |
| **Partition Key** | `idgroup` | Work is scoped to operational units, not tenants. |
| **ID Strategy** | Generate new UUIDs; store `legacy_merge_key` | Clean break, but with traceability. |
| **Mapping Granularity** | One unified task per legacy **parent row** | Matches current UI behavior; safer migration. |

### 2.4 Deliverables

- [ ] Mapping specification document (legacy column → new column).
- [ ] List of all endpoints to modify for dual-write.
- [ ] Rollback plan document.

---

## 3. Phase 1: Schema Deployment

**Duration:** 1 day
**Owner:** Backend Developer
**Risk Level:** 🟢 Low (Additive changes only)

### 3.1 Create Migrations

Execute in `core-2.0/database/migrations/`:

```bash
php artisan make:migration create_unified_tasks_table
php artisan make:migration create_task_locations_table
php artisan make:migration create_task_tags_table
php artisan make:migration create_task_assignments_table
php artisan make:migration create_task_resources_table
php artisan make:migration create_task_products_table
php artisan make:migration create_task_ext_nutritional_table
```

### 3.2 Migration Content (Key Points)

See `03_TechnicalDesign_TOBE.md` for full schema. Key requirements:

- `tasks.id` — BINARY(16) UUID.
- `tasks.legacy_merge_key` — TEXT, nullable (for mapping).
- `tasks.version` — INT DEFAULT 1 (for optimistic locking).
- All junction tables have composite unique keys.

### 3.3 Acceptance Criteria

- [ ] `php artisan migrate` runs cleanly on dev/staging.
- [ ] No changes to existing `task` or `spraying` tables.
- [ ] No existing endpoints are modified.

---

## 4. Phase 2: Dual-Write Implementation

**Duration:** 1-2 weeks
**Owner:** Backend Developer
**Risk Level:** 🟠 Medium (Performance impact on writes)

### 4.1 Create TaskService

Implement a new service that writes to the Model 3 schema:

```php
// app/Services/TaskService.php

class TaskService {
    public function createFromLegacyPayload(array $legacyPayload): Task {
        // Map legacy fields to new structure
        // Create tasks row
        // Create junction rows (assignments, resources, locations, tags)
        // Store legacy_merge_key for mapping
    }

    public function updateFromLegacyPayload(string $mergeKey, array $legacyPayload): void {
        // Find existing task by legacy_merge_key
        // Update core fields
        // Sync junction tables
    }

    public function deleteByLegacyMergeKey(string $mergeKey): void {
        // Find and soft-delete task by legacy_merge_key
    }
}
```

### 4.2 Implement Dual-Write in Controllers

Modify existing endpoints to write to **both** schemas:

```php
// ScheduleController.php

public function createSchedule(ScheduleCreateRequest $request) {
    DB::transaction(function () use ($request) {
        // 1. Execute legacy write (existing logic)
        $result = $this->scheduleRepository->createSchedule($request->validated());

        // 2. Execute TO-BE write (new logic)
        try {
            $this->taskService->createFromLegacyPayload($request->validated());
        } catch (\Exception $e) {
            Log::error('TO-BE write failed', ['error' => $e->getMessage()]);
            // Do NOT rollback — legacy is primary
        }

        return $result;
    });
}
```

### 4.3 Dual-Write Endpoints Checklist

| Endpoint | Controller Method | Status |
| :--- | :--- | :--- |
| `POST /createSchedule` | `createSchedule` | ☐ |
| `PUT /update-schedule` | `updateTask` | ☐ |
| `DELETE /schedule/{mergeKey}` | `deleteSchedule` | ☐ |
| `POST /createSprayingRecords` | `createSprayingRecords` | ☐ |
| `DELETE /deleteSprayingRecords` | `deleteSprayingRecords` | ☐ |

### 4.4 Acceptance Criteria

- [ ] Every new/updated/deleted task in Model 2 is mirrored to Model 3.
- [ ] Model 3 writes are logged (success/failure).
- [ ] Model 2 remains the primary read source.

---

## 5. Phase 3: Historical Backfill

**Duration:** 2-3 days
**Owner:** Backend Developer
**Risk Level:** 🟠 Medium (Large data volume)

### 5.1 Create Backfill Command

```bash
php artisan make:command BackfillUnifiedTasks
```

### 5.2 Algorithm

```php
// app/Console/Commands/BackfillUnifiedTasks.php

public function handle() {
    // 1. Get all distinct merge_keys from task table
    $mergeKeys = DB::table('task')
        ->select('merge_key')
        ->distinct()
        ->cursor();

    foreach ($mergeKeys as $row) {
        $this->processCluster($row->merge_key);
    }
}

private function processCluster(string $mergeKey) {
    // 2. Get all parent rows for this merge_key
    $parents = DB::table('task')
        ->where('merge_key', $mergeKey)
        ->whereNull('idparent_task')
        ->get();

    foreach ($parents as $parent) {
        // 3. Check if already migrated (idempotent)
        if ($this->alreadyMigrated($parent->idtask)) {
            continue;
        }

        // 4. Create unified task
        $task = $this->createUnifiedTask($parent, $mergeKey);

        // 5. Create assignments from child rows
        $this->migrateStaffAssignments($parent->idtask, $task->id);

        // 6. Create resources from child rows
        $this->migrateMachineResources($parent->idtask, $task->id);

        // 7. Create tags from taggable
        $this->migrateTags($parent->idtask, $task->id);
    }

    // 8. Migrate product links for merge_key
    $this->migrateProducts($mergeKey, $firstTaskId);
}
```

### 5.3 Mapping Reference

| Legacy Field | Target Field |
| :--- | :--- |
| `task.idtask` | `tasks.legacy_task_id` |
| `task.merge_key` | `tasks.legacy_merge_key` |
| `task.idsite` | `task_locations.site_id` |
| `task.idaction` | `tasks.action_id` |
| `task.estimate_time` | `tasks.planned_minutes` |
| `task.task_start` | `tasks.actual_start` |
| `task.running` | `tasks.status` (0=PLANNED, 1=RUNNING) |
| Child row with `idstaff` | `task_assignments` row |
| Child row with `idmachinery` | `task_resources` row |
| `taggable` rows | `task_tags` rows |
| `product_task` rows | `task_products` rows |

### 5.4 Migrate Spraying

```php
// For each spraying record:
// 1. Create tasks row with type_id = 'SPRAYING'
// 2. Create task_ext_nutritional row
// 3. Map locations, tags, products
```

### 5.5 Migrate Report Follow-ups

Follow-up actions are migrated from the separate `report_followup` table to the unified `tasks` table.

```php
// app/Console/Commands/BackfillFollowups.php

public function handle() {
    $followups = DB::table('report_followup')->cursor();

    foreach ($followups as $followup) {
        // 1. Check if already migrated (idempotent)
        if ($this->alreadyMigrated($followup->id)) {
            continue;
        }

        // 2. Create unified task with type_id = 'FOLLOWUP'
        $taskId = Str::uuid();

        DB::table('tasks')->insert([
            'id' => $taskId,
            'idgroup' => $this->getGroupFromReport($followup->report_id),
            'type_id' => 'FOLLOWUP',
            'action_id' => null,                    // Follow-ups don't have actions
            'title' => Str::limit($followup->description, 255),
            'planned_start' => $followup->due_date, // Due date becomes planned_start for calendar
            'planned_end' => $followup->due_date,
            'status' => $this->mapFollowupStatus($followup->status),
            'notes' => $followup->description,
            'created_at' => $followup->created_at,
            'updated_at' => now(),
            'created_by' => $followup->created_by,
            'legacy_followup_id' => $followup->id,  // For traceability
        ]);

        // 3. Create extension row with follow-up specific data
        DB::table('task_ext_followup')->insert([
            'task_id' => $taskId,
            'parent_report_id' => $followup->report_id,
            'parent_task_id' => $followup->task_id,
            'priority' => strtoupper($followup->priority),
            'due_date' => $followup->due_date,
            'completed_at' => $followup->completed_at,
            'completed_by' => $followup->completed_by,
        ]);

        // 4. Create assignment for the assignee
        if ($followup->assignee_id) {
            DB::table('task_assignments')->insert([
                'id' => Str::uuid(),
                'task_id' => $taskId,
                'user_id' => $followup->assignee_id,
                'role' => 'ASSIGNEE',
            ]);
        }

        Log::info("Migrated follow-up {$followup->id} to task {$taskId}");
    }
}

private function mapFollowupStatus(string $legacyStatus): string {
    return match($legacyStatus) {
        'open' => 'PLANNED',
        'in_progress' => 'RUNNING',
        'completed' => 'COMPLETED',
        'overdue' => 'PLANNED',  // Overdue is still an open task
        default => 'PLANNED',
    };
}
```

#### Follow-up Field Mapping

| Legacy Field (`report_followup`) | Target Field |
| :--- | :--- |
| `id` | `tasks.legacy_followup_id` (new compatibility field) |
| `report_id` | `task_ext_followup.parent_report_id` |
| `task_id` | `task_ext_followup.parent_task_id` |
| `description` | `tasks.notes` + `tasks.title` (truncated) |
| `assignee_id` | `task_assignments.user_id` (role = 'ASSIGNEE') |
| `due_date` | `tasks.planned_start` + `task_ext_followup.due_date` |
| `priority` | `task_ext_followup.priority` |
| `status` | `tasks.status` (mapped: open→PLANNED, etc.) |
| `completed_at` | `task_ext_followup.completed_at` |
| `completed_by` | `task_ext_followup.completed_by` |
| `created_at` | `tasks.created_at` |
| `created_by` | `tasks.created_by` |

### 5.6 Acceptance Criteria

- [ ] Command is idempotent (safe to re-run).
- [ ] Metrics output: total processed, total created, total skipped.
- [ ] All legacy `merge_key` values are represented in `tasks.legacy_merge_key`.

---

## 6. Phase 4: Validation & Read Switch

**Duration:** 1 week
**Owner:** QA + Backend Developer
**Risk Level:** 🟠 Medium (User-facing change)

### 6.1 Validation Queries

Run these on staging before switching reads:

```sql
-- Count comparison: Legacy vs Unified
SELECT 
    (SELECT COUNT(DISTINCT merge_key) FROM task) AS legacy_clusters,
    (SELECT COUNT(*) FROM tasks) AS unified_tasks;

-- Staff assignment comparison
SELECT 
    (SELECT COUNT(*) FROM task WHERE idstaff IS NOT NULL) AS legacy_staff_rows,
    (SELECT COUNT(*) FROM task_assignments) AS unified_assignments;

-- Product link comparison
SELECT
    (SELECT COUNT(*) FROM product_task) AS legacy_product_links,
    (SELECT COUNT(*) FROM task_products) AS unified_product_links;

-- Follow-up comparison
SELECT
    (SELECT COUNT(*) FROM report_followup) AS legacy_followups,
    (SELECT COUNT(*) FROM tasks WHERE type_id = 'FOLLOWUP') AS unified_followups;

-- Follow-up status verification
SELECT
    rf.status AS legacy_status,
    t.status AS unified_status,
    COUNT(*) AS count
FROM report_followup rf
JOIN tasks t ON t.legacy_followup_id = rf.id
GROUP BY rf.status, t.status;
```

### 6.2 Implement V3 Read Endpoint

```php
// app/Http/Controllers/TaskController.php (NEW)

public function index(Request $request) {
    $query = Task::query()
        ->where('idgroup', $request->idgroup)
        ->whereBetween('planned_start', [$request->start_date, $request->end_date]);

    if ($request->has('include')) {
        $query->with(explode(',', $request->include));
    }

    return TaskResource::collection($query->paginate());
}
```

### 6.3 Feature Flag Implementation

```php
// config/feature_flags.php
return [
    'use_unified_task_read' => env('FEATURE_UNIFIED_TASK_READ', false),
];

// In ScheduleController
public function getSchedulesForCalender(Request $request) {
    if (config('feature_flags.use_unified_task_read')) {
        return $this->taskService->getCalendarData($request);
    }
    // Legacy implementation
    return $this->scheduleRepository->getSchedulesForCalender(...);
}
```

### 6.4 Frontend Migration

1.  Add feature flag check in `web/src/store/schedule/index.js`.
2.  When flag is enabled, call `/api/v3/tasks` instead of legacy endpoints.
3.  Map V3 response to existing UI data structures.

### 6.5 Acceptance Criteria

- [ ] Validation queries show 100% parity.
- [ ] Feature flag can be toggled without deployment.
- [ ] UI renders correctly from V3 endpoint in staging.

---

## 7. Phase 5: Deprecation & Cleanup

**Duration:** Post-validation
**Owner:** Tech Lead
**Risk Level:** 🟡 Low (Cleanup only)

### 7.1 Disable Dual-Write

Once V3 reads are stable in production:

```php
// Remove dual-write logic from ScheduleController
// All writes now go directly to TaskService
```

### 7.2 Archive Legacy Tables

```sql
-- Create archive copies
CREATE TABLE archive_task AS SELECT * FROM task;
CREATE TABLE archive_spraying AS SELECT * FROM spraying;
CREATE TABLE archive_taggable AS SELECT * FROM taggable;
CREATE TABLE archive_product_task AS SELECT * FROM product_task;
CREATE TABLE archive_report_followup AS SELECT * FROM report_followup;

-- After validation period (30 days), drop legacy tables
DROP TABLE task;
DROP TABLE spraying;
DROP TABLE taggable;
DROP TABLE product_task;
DROP TABLE report_followup;
```

### 7.3 Remove Legacy Compatibility Fields

```bash
php artisan make:migration remove_legacy_fields_from_tasks
```

```php
Schema::table('tasks', function (Blueprint $table) {
    $table->dropColumn(['legacy_merge_key', 'legacy_task_id', 'legacy_spraying_id']);
});
```

### 7.4 Deprecate Core 1 Dependencies

Separate workstream (not included in this playbook). Track via Appendix A in `02_TechnicalAudit_ASIS.md`.

---

## 8. Rollback Plan

If critical issues are discovered:

### 8.1 Quick Rollback (Within Dual-Write Phase)
1.  Disable feature flag: `FEATURE_UNIFIED_TASK_READ=false`.
2.  Remove dual-write logic (revert to pure legacy writes).
3.  Truncate Model 3 tables (data is already in legacy).

### 8.2 Full Rollback (After Read Switch)
1.  Restore from archive tables.
2.  Redeploy legacy controllers.
3.  Root cause analysis.

---

## 9. Checklist Summary

| Phase | Task | Owner | Status |
| :--- | :--- | :--- | :--- |
| **0** | Document merge_key algorithm | Tech Lead | ☐ |
| **0** | Create mapping specification | Tech Lead | ☐ |
| **1** | Create Model 3 migrations | Backend | ☐ |
| **1** | Run migrations on staging | DevOps | ☐ |
| **2** | Implement TaskService | Backend | ☐ |
| **2** | Add dual-write to Schedule endpoints | Backend | ☐ |
| **2** | Add dual-write to Spraying endpoints | Backend | ☐ |
| **2** | Add dual-write to Follow-up endpoints | Backend | ☐ |
| **3** | Implement backfill command (Tasks) | Backend | ☐ |
| **3** | Implement backfill command (Follow-ups) | Backend | ☐ |
| **3** | Run backfill on staging | DevOps | ☐ |
| **3** | Validate data parity (Tasks) | QA | ☐ |
| **3** | Validate data parity (Follow-ups) | QA | ☐ |
| **4** | Implement V3 read endpoint | Backend | ☐ |
| **4** | Add feature flag | Backend | ☐ |
| **4** | Update frontend for V3 | Frontend | ☐ |
| **4** | Enable flag in staging | DevOps | ☐ |
| **4** | UAT validation | QA | ☐ |
| **4** | Enable flag in production | DevOps | ☐ |
| **5** | Disable dual-write | Backend | ☐ |
| **5** | Archive legacy tables | DBA | ☐ |
| **5** | Remove legacy compatibility fields | Backend | ☐ |

---

## 10. Out of Scope (This Migration)

The following modules are **explicitly excluded** from the Model 3 migration to limit scope and risk:

| Module | Current Tables | Reason for Exclusion |
| :--- | :--- | :--- |
| **Action Planner** | `action_plan` | Annual/strategic planning — different lifecycle than daily tasks |
| **Daily Routine** | `daily_routine`, `daily_routine_tasks`, `daily_routine_site`, `daily_routine_staff` | Template system for recurring tasks — separate concern |
| **Spraying Routine** | `spraying_routine_*` | Template system for recurring spraying — separate concern |
| **Fertilisation Planner** | `fertilization_plan_template`, `fertilization_plan` | Nutrient planning — integrates with Inventory, not Schedule |
| **Budget Planner** | `budget_plan`, `actual_order_plan` | Financial planning — out of domain |

### Why Not Migrate Planner Modules?

1. **Different Purpose:** Planners are for *strategic/annual planning*, not *daily execution*
2. **Stable & Functional:** Current Planner tables work correctly — no urgent technical debt
3. **Scope Creep Risk:** Adding Planner would 2-3x the migration effort
4. **Dependency Chain:** Routines generate tasks → must ensure Model 3 tasks work first

### Future Phase Consideration

After Model 3 is stable in production, a **Phase 2 migration** could unify Planner modules:
- Action Planner → `tasks` with `type_id = 'ACTION_PLAN'`
- Daily Routine → `task_templates` + recurrence rules
- Spraying Routine → `task_templates` with `type_id = 'ROUTINE_SPRAYING'`

> **See:** `03_TechnicalDesign_TOBE.md` Section 9 (Future Roadmap) and ADR-006 for architectural details.
> **See:** `docu/features/planner/` for current Planner documentation.
