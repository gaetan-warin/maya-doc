# Technical Audit Document (AS-IS Analysis)
## Maya — Task Management & Scheduling Module

| Document Details | |
| :--- | :--- |
| **Project** | Maya Core 2.0 |
| **Module** | Scheduling & Task Management |
| **Version** | 1.0 |
| **Status** | Authoritative Reference |
| **Target Audience** | Developers, DevOps, Technical Leads |

---

## 1. Executive Summary

This document provides a **brutally honest** technical assessment of the current ("AS-IS") Scheduling system. It is intended to help developers understand the existing implementation, its quirks, and its known problems before making changes or executing a migration.

### Key Findings

| Area | Assessment | Risk Level |
| :--- | :--- | :--- |
| **Data Model** | Highly denormalized; single table serves multiple purposes. | 🔴 High |
| **Grouping Mechanism** | `merge_key` (LONGTEXT) used as primary grouping key. | 🔴 High |
| **System Coupling** | Inventory, TOIL, and Products are tightly coupled to `merge_key`. | 🟠 Medium |
| **Legacy Dependencies** | Frontend still relies on Core 1 (OData) for critical data. | 🟠 Medium |
| **Performance** | Schedule page can trigger 20+ API calls; degrades with scale. | 🟠 Medium |
| **Event System** | TOIL automation listener is **not wired** by default. | 🟡 Low |

---

## 2. System Architecture Overview

### 2.1 Component Map

```
┌─────────────────────────────────────────────────────────────────┐
│                         FRONTEND (Vue.js)                       │
│    web/src/store/schedule/index.js                              │
│    web/src/composables/formatting/schedule/scheduleSaveTasks.js │
└───────────────────────────┬─────────────────────────────────────┘
                            │
          ┌─────────────────┴─────────────────┐
          │                                   │
          ▼                                   ▼
┌─────────────────────┐             ┌─────────────────────┐
│   CORE 2.0 (REST)   │             │   CORE 1 (OData)    │
│   /api/v2/...       │             │   /api/odata/v1/... │
│                     │             │                     │
│  ScheduleController │             │  Actions, Staff,    │
│  SprayingController │             │  TOIL, Fleet,       │
│                     │             │  Product Lookups    │
└─────────────────────┘             └─────────────────────┘
          │                                   │
          └─────────────────┬─────────────────┘
                            │
                            ▼
                  ┌─────────────────────┐
                  │      DATABASE       │
                  │   (MySQL / MariaDB) │
                  │                     │
                  │   task, spraying,   │
                  │   taggable,         │
                  │   product_task,     │
                  │   inventory_ledger  │
                  └─────────────────────┘
```

### 2.2 Codebase Locations

| Layer | Path | Responsibility |
| :--- | :--- | :--- |
| **Backend Controller** | `core-2.0/app/Http/Controllers/ScheduleController.php` | CRUD endpoints for Tasks. |
| **Backend Repository** | `core-2.0/app/Repositories/ScheduleRepository.php` | Database access, complex queries. |
| **Backend Service** | `core-2.0/app/Services/ScheduleService.php` | Business logic, inventory side-effects. |
| **Frontend Store** | `web/src/store/schedule/index.js` | API calls to backend. |
| **Frontend Logic** | `web/src/composables/formatting/schedule/scheduleSaveTasks.js` | Payload construction, `merge_key` generation. |

---

## 3. Data Model (Legacy Schema)

The current schema is characterized by **row duplication** and a **string-based grouping key**.

### 3.1 Core Table: `task`

This table serves **multiple purposes**, which is a primary source of technical debt.

| Column | Type | Purpose |
| :--- | :--- | :--- |
| `idtask` | BINARY(16) | Primary Key (UUID). |
| `idparent_task` | BINARY(16) | NULL for "parent" rows; points to parent for "child" rows. |
| `merge_key` | LONGTEXT | The grouping key for a Task Cluster. |
| `idgroup` | BINARY(16) | **Partition Key** (Tenant's Operational Unit). |
| `idsite` | BINARY(16) | The Site where work is performed. |
| `idaction` | BINARY(16) | The Action (type of work). |
| `idstaff` | BINARY(16) | Set on **child rows** to represent staff assignment. |
| `idmachinery` | BINARY(16) | Set on **child rows** to represent machine assignment. |
| `task_start` | TIMESTAMP | Actual start time. |
| `task_end` | TIMESTAMP | Actual end time. |
| `estimate_time` | INT | Planned duration in minutes. |
| `running` | TINYINT | Status flag (0=Planned, 1=Running). |

> **⚠️ Critical Insight:** A single user-facing "Task" is represented by **multiple rows**:
> - **1 Parent Row:** Contains the main task data (Site, Action, Times).
> - **N Child Rows (Staff):** One row per assigned staff member (`idstaff` set).
> - **M Child Rows (Machine):** One row per assigned machine (`idmachinery` set).

### 3.2 Grouping Table: `taggable`

Links Areas/Tags to Tasks.

| Column | Type | Purpose |
| :--- | :--- | :--- |
| `taggable_id` | BINARY(16) | The Task ID (can be parent or child). |
| `tag_id` | BINARY(16) | The Area/Tag ID. |

> **⚠️ Issue:** Tags are duplicated across parent and all child rows.

### 3.3 Product Link Table: `product_task`

Links Products to Task Clusters.

| Column | Type | Purpose |
| :--- | :--- | :--- |
| `merge_key_hash` | VARCHAR(64) | `SHA256(merge_key)` for indexing. |
| `idproduct` | BINARY(16) | The Product ID. |
| `quantity` | DECIMAL | The quantity to consume. |

> **⚠️ Critical Coupling:** Products are linked by `merge_key`, **not** by `idtask`. If the `merge_key` changes, product links must be migrated.

### 3.4 Inventory Ledger: `inventory_ledger`

Records stock movements.

| Column | Type | Purpose |
| :--- | :--- | :--- |
| `merge_key` | LONGTEXT | The Task Cluster's merge_key. |
| `idspraying` | BINARY(16) | For Spraying records. |
| `idproduct` | BINARY(16) | The Product. |
| `quantity` | DECIMAL | The amount consumed (negative) or refunded (positive). |

### 3.5 Spraying Table: `spraying`

A **completely separate table** for Nutritional Inputs.

| Column | Type | Purpose |
| :--- | :--- | :--- |
| `idspraying` | BINARY(16) | Primary Key. |
| `idsite` | BINARY(16) | Location. |
| `nozzle_type`, `speed`, `rpm`, `pressure` | Various | Calibration data. |

> **⚠️ Siloed Data:** Tasks and Spraying are in separate tables, making unified reporting difficult.

---

## 4. The `merge_key` Problem

The `merge_key` is the system's central (and most problematic) concept.

### 4.1 What It Is
A long string generated by the **Frontend** that groups multiple database rows into a single logical Task.

### 4.2 How It's Generated
The frontend (`scheduleSaveTasks.js`) concatenates:
- All selected Site IDs (dashes removed).
- All selected Hole IDs (dashes removed).
- The Action ID (dashes removed).
- The date in `YYYYMMDD` format.
- A random salt / timestamp.

### 4.3 Why It's Problematic

| Issue | Impact |
| :--- | :--- |
| **LONGTEXT Column** | Cannot be directly indexed. Queries use `MD5()` or `SHA256()` hashing workarounds. |
| **Generated by Frontend** | The backend does not control the grouping logic. |
| **Mutable** | If the user edits a Task (changes Site or Action), the `merge_key` **changes**. |
| **Critical Dependency** | `product_task` and `inventory_ledger` are keyed by `merge_key`. |

### 4.4 Consequence
If a `merge_key` changes mid-edit:
1.  The backend must **migrate** all `product_task` links to the new key.
2.  The backend must update `inventory_ledger` references.
3.  If any step fails, data becomes fragmented (historical audit trail is broken).

---

## 5. API Surface (Current Endpoints)

### 5.1 Schedule Endpoints (Core 2.0)

| Method | Endpoint | Controller Method | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/createSchedule` | `createSchedule` | Creates a Task Cluster. |
| `PUT` | `/update-schedule?mergeKey=...` | `updateTask` | Updates a Task Cluster. |
| `DELETE` | `/schedule/{mergeKey}` | `deleteSchedule` | Deletes entire Task Cluster. |
| `DELETE` | `/schedule/task/{taskId}` | `deleteScheduleSingleTask` | Deletes one parent + children. |
| `GET` | `/schedules?idtenant=...&start_date=...&end_date=...` | `getSchedulesForCalender` | Calendar view (Tasks + Spraying). |
| `POST` | `/tasks/products` | `addProductsToTasks` | Links products to a `merge_key`. |
| `PUT` | `/tasks/products` | `updateProductsToTasks` | Updates product quantities. |

### 5.2 Spraying Endpoints (Core 2.0)

| Method | Endpoint | Controller Method | Description |
| :--- | :--- | :--- | :--- |
| `GET` | `/getSprayingRecords` | `getSprayingRecords` | Fetches spraying data. |
| `POST` | `/createSprayingRecords` | `createSprayingRecords` | Creates a spraying record. |
| `DELETE` | `/deleteSprayingRecords` | `deleteSprayingRecords` | Deletes a spraying record. |

### 5.3 Legacy Endpoints (Core 1 / OData)

The frontend still calls these via `{ useLegacyApi: true }`:

| Module | Endpoints |
| :--- | :--- |
| **Schedule** | `POST /updateStaff`, `POST /action`, `PATCH /action(...)` |
| **TOIL** | `GET /toil?$filter=...`, `PATCH /toil(...)` |
| **Spraying Lookups** | `GET /product_category?...`, `GET /unit?...` |
| **Fleet** | `GET /getMachineData(...)` |
| **Weather** | `GET /site?$expand=devices`, `POST /weatherChartData` |

---

## 6. Known Pitfalls & Technical Debt

### 6.1 One Logical Task = Multiple DB Rows
Developers querying the `task` table will see apparent "duplicates". This is **by design**—each staff/machine assignment creates a child row.

### 6.2 `merge_key` Changes Can "Split" History
If the frontend regenerates the `merge_key` on edit, historical links (products, inventory) may become orphaned or require complex migration logic.

### 6.3 Calendar Grouping Uses `createdon`, Not `task_start`
Some queries group tasks by `DATE(createdon)` instead of the planned date. This can cause confusion if a task is created on Day 1 for Day 2.

### 6.4 TOIL Automation Is Not Active
The codebase contains `CreateToilRecordListener`, but `EventServiceProvider` has an **empty** `$listen` array. Automatic TOIL creation does not run unless manually wired.

### 6.5 Frontend Drives Merge Key Logic
The backend does not regenerate or validate the `merge_key`. It trusts the frontend payload. This is a potential source of bugs.

---

## 7. Performance Concerns

Based on Notion documentation ("Schedule Page Revamp"):

| Issue | Detail |
| :--- | :--- |
| **20+ API Calls** | The Schedule page triggers many sequential requests on load. |
| **Degradation at Scale** | Performance degrades significantly with 20+ staff and 100+ tasks. |
| **N+1 Queries** | Some repository methods fetch related data in loops. |

---

## 8. Recommendations (Pre-Migration)

Before executing a migration to the "TO-BE" (Model 3) architecture:

1.  **Freeze the `merge_key` Generation Logic:** Document the exact algorithm and ensure backend validation.
2.  **Audit `inventory_ledger` Links:** Identify any orphaned `merge_key` references.
3.  **Wire the TOIL Listener (Optional):** If TOIL automation is required, register the listener.
4.  **Profile the Schedule Page:** Identify the top 5 slowest API calls for optimization.
