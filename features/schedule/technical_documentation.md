# Schedule Management
## Technical Design Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-TDS-SCHED-003 |
| **Version** | 3.1 (Deep Analysis & Schema) |
| **Last Updated** | December 31, 2025 |
| **Status** | Approved |
| **Owner** | Engineering Team |
| **Classification** | Internal - Technical |

---

## 1. System Architecture (Model 3)

### 1.1 Architectural Shift
The "Model 3" architecture moves from a **Denormalized Flat Table** to a **Star Schema** approach.
*   **Central Fact:** `tasks` table.
*   **Dimensions:** `task_assignments`, `task_resources`, `task_locations`.
*   **Extensions:** `task_ext_nutritional`.

---

## 2. Database Schema (Target)

### 2.1 Core Entity: `tasks`
| Column | Type | Description |
|:---|:---|:---|
| `id` | BINARY(16) | **Immutable PK (UUID)**. |
| `type_id` | VARCHAR(32) | `GENERAL`, `SPRAYING`, `LEAVE`. |
| `status` | ENUM | `DRAFT`, `PLANNED`, `RUNNING`, `COMPLETED`, `CANCELLED`. |
| `planned_start` | DATETIME | ISO 8601. |
| `actual_start` | DATETIME | Logged punch-in time. |
| `version` | INT | **Optimistic Locking Counter**. |
| `legacy_merge_key` | TEXT | **Transition Field**. Preserves old string grouping during migration. |

### 2.2 Extension: `task_ext_nutritional` (Spraying)
Stores the polymorphic data specific to chemical applications.
| Column | Type | Description |
|:---|:---|:---|
| `task_id` | BINARY(16) | FK to `tasks.id`. |
| `nozzle_type_id` | BINARY(16) | Calibration metadata. |
| `speed` | DECIMAL | Application speed (Km/h). |
| `pressure` | DECIMAL | Application pressure (Bar). |
| `compliance_data` | LONGTEXT | JSON blob for variable regulatory fields. |

### 2.3 Junction Tables
| Table | Columns | Purpose |
|:---|:---|:---|
| `task_assignments` | `task_id`, `user_id`, `role` | Links Staff. Role: `LEAD`, `WORKER`. |
| `task_resources` | `task_id`, `resource_id` | Links Machines/Implements. |
| `task_locations` | `task_id`, `site_id` | Links Spatial Data (Multi-site support). |

---

## 3. Migration Strategy (Legacy Bridge)

### 3.1 Migration SQL Logic
The following logic defines how we move from the legacy `task` table to Model 3.

```sql
INSERT INTO tasks (
    id, idgroup, type_id, action_id, planned_minutes, 
    status, legacy_task_id, legacy_merge_key
)
SELECT 
    UUID(),                     -- Generate new UUID
    t.idgroup, 
    'GENERAL',                  -- Default Type
    t.idaction, 
    t.estimate_time,
    CASE 
        WHEN t.task_end IS NOT NULL THEN 'COMPLETED'
        WHEN t.running = 1 THEN 'RUNNING'
        ELSE 'PLANNED' 
    END AS status,
    t.idtask,                   -- Keep old ID for reference
    t.merge_key                 -- Preserve grouping
FROM task t 
WHERE t.idparent_task IS NULL;  -- Only migrate Parents (Children become Assignments)
```

### 3.2 Dual Write Strategy
During the transition phase (Phase 2), the application will:
1.  Read from `tasks` (New).
2.  Write to **Both** `tasks` and `legacy_schedule` (Old).
3.  This ensures legacy Mobile Apps (which read the old table) continue to function until updated.

---

## 4. API Contract (V3)

### 4.1 Unified Write Endpoint
`POST /api/v3/tasks`
*   **Concept:** A single transactional channel. No more "Create Task" then "Add Staff".
*   **Payload:**
    ```json
    {
      "type": "SPRAYING",
      "planned_start": "2025-05-01T08:00:00Z",
      "locations": ["uuid-site-1", "uuid-site-2"],
      "assignments": [
        { "user_id": "uuid-john", "role": "LEAD" },
        { "user_id": "uuid-jane", "role": "WORKER" }
      ],
      "extension": {
        "nozzle_id": "uuid-red-nozzle",
        "pressure": 3.5
      }
    }
    ```

### 4.2 Optimistic Concurrency
*   All `PATCH` requests **MUST** include the `version` field.
*   **Server Logic:**
    ```php
    if ($request->version != $dbTask->version) {
        abort(409, "Task has been modified by another user.");
    }
    ```

---

## 5. Technical Pitfalls & Debts

### 5.1 TOIL Integration Gap
*   **Issue:** The `CreateToilRecordListener` exists in the codebase but was found to be **inactive** (not registered in `EventServiceProvider`).
*   **Result:** Schedule completion does not currently reliably generate TOIL records.
*   **Fix:** Must be re-wired in the V3 `TaskCompleted` event.

### 5.2 Frontend Key Generation
*   **Strict Ban:** The Legacy system allowed the frontend to generate `merge_key` strings.
*   **New Rule:** The Frontend is strictly forbidden from minting IDs or Keys. It must accept server-provided UUIDs.

---

**Document End**
