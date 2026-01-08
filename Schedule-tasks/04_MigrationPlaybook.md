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
| Which UI entry points create Follow-ups? | Reports page → `POST /report-and-follow-up/{idTenant}` |
| Which backend endpoints write data? | `ScheduleController`, `SprayingController`, `ReportController` |

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
php artisan make:migration create_task_ext_followup_table
php artisan make:migration create_task_images_table
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
| `POST /report-and-follow-up/{idTenant}` | `createReportAndFollowUp` | ☐ |
| `PUT /update-report-and-follow-up/{idReport}` | `updateReportAndFollowUp` | ☐ |
| `PUT /update-Followup-schedule-status/{followupId}` | `updateFollowupStatusBySchedule` | ☐ |
| `POST /change-followup-status/{idReport}` | `updateFollowupStatus` | ☐ |

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

### 5.5 Migrate Follow-ups

```php
// app/Console/Commands/BackfillUnifiedTasks.php (continued)

private function migrateFollowups() {
    $followups = DB::table('report_follow_up')
        ->join('report', 'report_follow_up.idreport', '=', 'report.idreport')
        ->leftJoin('incident', 'report.idincident', '=', 'incident.idincident')
        ->select(
            'report_follow_up.*',
            'report.idgroup',
            'report.idsite',
            'incident.name as incident_name'
        )
        ->cursor();

    foreach ($followups as $followup) {
        // 1. Check if already migrated (idempotent)
        if ($this->followupAlreadyMigrated($followup->idreport_follow_up)) {
            continue;
        }

        // 2. Create unified task
        $taskId = $this->createFollowupTask($followup);

        // 3. Create extension record
        $this->createFollowupExtension($taskId, $followup);

        // 4. Create staff assignment (if assigned)
        if ($followup->iduser) {
            $this->createFollowupAssignment($taskId, $followup->iduser);
        }

        // 5. Create location (from parent report)
        if ($followup->idsite) {
            $this->createFollowupLocation($taskId, $followup->idsite);
        }

        // 6. Migrate images
        $this->migrateFollowupImages($taskId, $followup->idreport_follow_up);
    }
}

private function createFollowupTask($followup): string {
    $taskId = Str::orderedUuid()->toString();

    DB::table('tasks')->insert([
        'id' => Str::uuid()->getBytes(),
        'idgroup' => $followup->idgroup,
        'type_id' => 'FOLLOWUP',
        'title' => 'Follow-up: ' . ($followup->incident_name ?? 'Incident Report'),
        'planned_start' => $followup->date,
        'planned_minutes' => $followup->total_duration
            ? (int) (strtotime($followup->total_duration) - strtotime('00:00:00')) / 60
            : null,
        'status' => $followup->status === 'completed' ? 'COMPLETED' : 'PLANNED',
        'notes' => $followup->comment,
        'version' => 1,
        'created_at' => $followup->created_at,
        'updated_at' => $followup->updated_at,
    ]);

    return $taskId;
}
```

### 5.6 Follow-up Mapping Reference

| Legacy Field | Target Field |
| :--- | :--- |
| `report_follow_up.idreport_follow_up` | `task_ext_followup.legacy_followup_id` |
| `report_follow_up.idreport` | `task_ext_followup.report_id` |
| `report_follow_up.iduser` | `task_assignments.user_id` |
| `report_follow_up.date` | `tasks.planned_start` |
| `report_follow_up.comment` | `tasks.notes` |
| `report_follow_up.total_duration` | `tasks.planned_minutes` (converted) |
| `report_follow_up.total_cost` | `task_ext_followup.total_cost` |
| `report_follow_up.status` | `task_ext_followup.followup_status` |
| `report_follow_up_image.image_url` | `task_images.image_url` |

### 5.7 Acceptance Criteria

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
    (SELECT COUNT(*) FROM tasks WHERE type_id = 'GENERAL') AS unified_general_tasks;

-- Staff assignment comparison
SELECT
    (SELECT COUNT(*) FROM task WHERE idstaff IS NOT NULL) AS legacy_staff_rows,
    (SELECT COUNT(*) FROM task_assignments) AS unified_assignments;

-- Product link comparison
SELECT
    (SELECT COUNT(*) FROM product_task) AS legacy_product_links,
    (SELECT COUNT(*) FROM task_products) AS unified_product_links;

-- Spraying comparison
SELECT
    (SELECT COUNT(*) FROM spraying) AS legacy_spraying,
    (SELECT COUNT(*) FROM tasks WHERE type_id = 'SPRAYING') AS unified_spraying;

-- Follow-up comparison
SELECT
    (SELECT COUNT(*) FROM report_follow_up) AS legacy_followups,
    (SELECT COUNT(*) FROM tasks WHERE type_id = 'FOLLOWUP') AS unified_followups;

-- Follow-up images comparison
SELECT
    (SELECT COUNT(*) FROM report_follow_up_image) AS legacy_followup_images,
    (SELECT COUNT(*) FROM task_images WHERE image_type = 'evidence') AS unified_followup_images;
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
CREATE TABLE archive_report_follow_up AS SELECT * FROM report_follow_up;
CREATE TABLE archive_report_follow_up_image AS SELECT * FROM report_follow_up_image;

-- After validation period (30 days), drop legacy tables
DROP TABLE task;
DROP TABLE spraying;
DROP TABLE taggable;
DROP TABLE product_task;
DROP TABLE report_follow_up_image;
DROP TABLE report_follow_up;
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
| **2** | Implement FollowupExtensionHandler | Backend | ☐ |
| **3** | Implement backfill command | Backend | ☐ |
| **3** | Backfill follow-up records | Backend | ☐ |
| **3** | Run backfill on staging | DevOps | ☐ |
| **3** | Validate data parity | QA | ☐ |
| **4** | Implement V3 read endpoint | Backend | ☐ |
| **4** | Add feature flag | Backend | ☐ |
| **4** | Update frontend for V3 | Frontend | ☐ |
| **4** | Enable flag in staging | DevOps | ☐ |
| **4** | UAT validation | QA | ☐ |
| **4** | Enable flag in production | DevOps | ☐ |
| **5** | Disable dual-write | Backend | ☐ |
| **5** | Archive legacy tables | DBA | ☐ |
| **5** | Remove legacy compatibility fields | Backend | ☐ |
