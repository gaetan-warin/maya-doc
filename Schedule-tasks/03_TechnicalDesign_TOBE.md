# Technical Design Document (TO-BE Architecture)
## Maya — Unified Task Model (Model 3)

| Document Details | |
| :--- | :--- |
| **Project** | Maya Core 2.0 |
| **Module** | Scheduling & Task Management |
| **Version** | 1.0 |
| **Status** | Target Architecture (North Star) |
| **Target Audience** | Architects, Senior Developers, Tech Leads |

---

## 1. Executive Summary

This document defines the **target architecture** ("Model 3") for the Maya Scheduling module. It represents the "North Star" that all development efforts should move towards.

### Design Goals

| Goal | Description |
| :--- | :--- |
| **Unification** | A single `tasks` table for all scheduled work (including Spraying). |
| **Normalization** | Proper junction tables for Staff, Machines, Locations, and Products. |
| **Elimination of `merge_key`** | Replace string-based grouping with proper relational IDs. |
| **Extensibility** | Support for new task types via extension tables (`task_ext_*`). |
| **Performance** | Read-side materialization for fast calendar queries. |
| **Correctness** | Optimistic locking to prevent concurrent edit conflicts. |

---

## 2. Architectural Principles

### 2.1 One Entity, One Row
In the TO-BE model, a single user-facing "Task" is represented by **one row** in the `tasks` table. Assignments and resources are linked via junction tables, not duplicated rows.

### 2.2 Frontend Agnosticism
The backend becomes the **source of truth** for task identity. The frontend no longer generates the grouping key.

### 2.3 Type Discrimination
A single `tasks` table supports multiple task types (General, Spraying, Leave) via a `type_id` discriminator. Domain-specific fields live in extension tables.

---

## 3. Target Data Schema

### 3.1 Core Entity: `tasks`

The single source of truth for all scheduled work.

```sql
CREATE TABLE tasks (
    id              BINARY(16) PRIMARY KEY,      -- UUID
    idgroup         BINARY(16) NOT NULL,         -- Partition Key (Operational Unit)
    type_id         VARCHAR(32) NOT NULL,        -- e.g., 'GENERAL', 'SPRAYING', 'LEAVE'
    action_id       BINARY(16),                  -- FK to actions table
    title           VARCHAR(255),                -- Display name
    planned_start   DATETIME,                    -- Planned start time
    planned_end     DATETIME,                    -- Planned end time
    planned_minutes INT,                         -- Estimated duration
    actual_start    DATETIME,                    -- Recorded start time
    actual_end      DATETIME,                    -- Recorded end time
    status          ENUM('DRAFT', 'PLANNED', 'RUNNING', 'COMPLETED', 'CANCELLED'),
    notes           TEXT,
    parameters      LONGTEXT,                    -- Flexible metadata (JSON string)
    version         INT DEFAULT 1,               -- Optimistic locking
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_by      BINARY(16),                  -- FK to users

    -- Migration Compatibility (remove after deprecation)
    legacy_merge_key    TEXT,
    legacy_task_id      BINARY(16),
    legacy_spraying_id  BINARY(16),

    INDEX idx_group_date (idgroup, planned_start),
    INDEX idx_legacy_merge (legacy_merge_key(255))
);
```

### 3.2 Locations: `task_locations`

Supports multi-site tasks with proper relational links.

```sql
CREATE TABLE task_locations (
    id          BINARY(16) PRIMARY KEY,
    task_id     BINARY(16) NOT NULL,             -- FK to tasks
    site_id     BINARY(16) NOT NULL,             -- FK to sites
    hole_id     BINARY(16),                      -- FK to holes (optional)
    UNIQUE KEY uk_task_site_hole (task_id, site_id, hole_id)
);
```

### 3.3 Areas/Tags: `task_tags`

Many-to-many relationship between Tasks and Areas.

```sql
CREATE TABLE task_tags (
    task_id     BINARY(16) NOT NULL,             -- FK to tasks
    tag_id      BINARY(16) NOT NULL,             -- FK to tags
    PRIMARY KEY (task_id, tag_id)
);
```

### 3.4 Staff Assignments: `task_assignments`

Replaces the "child row per staff" pattern.

```sql
CREATE TABLE task_assignments (
    id              BINARY(16) PRIMARY KEY,
    task_id         BINARY(16) NOT NULL,         -- FK to tasks
    user_id         BINARY(16) NOT NULL,         -- FK to users (staff)
    role            VARCHAR(32) DEFAULT 'WORKER', -- 'LEAD', 'SUPPORT', 'OPERATOR'
    is_overtime     BOOLEAN DEFAULT FALSE,
    planned_minutes INT,
    actual_minutes  INT,
    UNIQUE KEY uk_task_user (task_id, user_id)
);
```

### 3.5 Resources: `task_resources`

Replaces the "child row per machine" pattern.

```sql
CREATE TABLE task_resources (
    id              BINARY(16) PRIMARY KEY,
    task_id         BINARY(16) NOT NULL,         -- FK to tasks
    resource_id     BINARY(16) NOT NULL,         -- FK to machines
    resource_type   VARCHAR(32) DEFAULT 'MACHINE', -- 'IMPLEMENT', 'VEHICLE'
    UNIQUE KEY uk_task_resource (task_id, resource_id)
);
```

### 3.6 Products: `task_products`

Replaces `product_task` with proper FK relationships.

```sql
CREATE TABLE task_products (
    id          BINARY(16) PRIMARY KEY,
    task_id     BINARY(16) NOT NULL,             -- FK to tasks
    product_id  BINARY(16) NOT NULL,             -- FK to products
    quantity    DECIMAL(10,2) NOT NULL,
    unit        VARCHAR(32),
    UNIQUE KEY uk_task_product (task_id, product_id)
);
```

### 3.7 Extension: `task_ext_nutritional` (Spraying)

Domain-specific fields for Nutritional Inputs.

```sql
CREATE TABLE task_ext_nutritional (
    task_id         BINARY(16) PRIMARY KEY,      -- FK to tasks
    nozzle_type_id  BINARY(16),
    spacing_id      BINARY(16),
    speed           DECIMAL(5,2),
    rpm             INT,
    pressure        DECIMAL(5,2),
    temperature     DECIMAL(4,1),
    wind_speed      DECIMAL(4,1),
    compliance_data LONGTEXT                     -- Flexible for H&S fields (JSON string)
);
```

---

## 4. Entity Relationship Diagram (ERD)

```
                                    ┌─────────────────┐
                                    │     tasks       │
                                    │─────────────────│
                                    │ id (PK)         │
                                    │ idgroup (FK)    │
                                    │ type_id         │
                                    │ action_id (FK)  │
                                    │ status          │
                                    │ version         │
                                    └────────┬────────┘
                                             │
         ┌───────────────┬───────────────┬───┴───┬───────────────┬───────────────┐
         │               │               │       │               │               │
         ▼               ▼               ▼       ▼               ▼               ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ task_locations  │ │ task_tags       │ │ task_assignments│ │ task_resources  │ │ task_products   │
│─────────────────│ │─────────────────│ │─────────────────│ │─────────────────│ │─────────────────│
│ task_id (FK)    │ │ task_id (FK)    │ │ task_id (FK)    │ │ task_id (FK)    │ │ task_id (FK)    │
│ site_id (FK)    │ │ tag_id (FK)     │ │ user_id (FK)    │ │ resource_id(FK) │ │ product_id (FK) │
│ hole_id (FK)    │ │                 │ │ role            │ │ resource_type   │ │ quantity        │
└─────────────────┘ └─────────────────┘ │ is_overtime     │ └─────────────────┘ └─────────────────┘
                                        └─────────────────┘

         ┌───────────────────────────────┐
         │     task_ext_nutritional      │
         │───────────────────────────────│
         │ task_id (PK/FK)               │
         │ nozzle_type_id, speed, rpm... │
         └───────────────────────────────┘
```

---

## 5. API Design (V3)

The TO-BE API should be RESTful, predictable, and support eager loading.

### 5.1 Endpoints

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/api/v3/tasks` | List tasks with filters and includes. |
| `GET` | `/api/v3/tasks/{id}` | Get a single task with all relations. |
| `POST` | `/api/v3/tasks` | Create a new task (all relations in one request). |
| `PATCH` | `/api/v3/tasks/{id}` | Update a task (supports optimistic locking via `version`). |
| `DELETE` | `/api/v3/tasks/{id}` | Delete a task. |

### 5.2 Query Parameters

| Parameter | Example | Description |
| :--- | :--- | :--- |
| `idgroup` | `?idgroup=...` | **Required.** Filter by operational unit. |
| `start_date` | `?start_date=2025-01-01` | Filter by planned start date. |
| `end_date` | `?end_date=2025-01-31` | Filter by planned end date. |
| `include` | `?include=assignments,resources,locations,extension` | Eager load relations. |
| `type_id` | `?type_id=SPRAYING` | Filter by task type. |

### 5.3 Request Payload (Create)

```json
{
  "idgroup": "uuid",
  "type_id": "GENERAL",
  "action_id": "uuid",
  "title": "Morning Mowing",
  "planned_start": "2025-01-15T08:00:00Z",
  "planned_end": "2025-01-15T11:00:00Z",
  "planned_minutes": 180,
  "notes": "Focus on greens.",
  "locations": [
    { "site_id": "uuid", "hole_id": "uuid" }
  ],
  "tags": ["uuid", "uuid"],
  "assignments": [
    { "user_id": "uuid", "role": "LEAD" },
    { "user_id": "uuid", "role": "WORKER" }
  ],
  "resources": [
    { "resource_id": "uuid", "resource_type": "MACHINE" }
  ],
  "products": [
    { "product_id": "uuid", "quantity": 10.5, "unit": "kg" }
  ],
  "extension": {
    "nozzle_type_id": "uuid",
    "speed": 8.5
  }
}
```

### 5.4 Response Payload

```json
{
  "id": "uuid",
  "idgroup": "uuid",
  "type_id": "GENERAL",
  "status": "PLANNED",
  "version": 1,
  "title": "Morning Mowing",
  "planned_start": "2025-01-15T08:00:00Z",
  "locations": [...],
  "assignments": [...],
  "resources": [...],
  "products": [...],
  "extension": {...},
  "created_at": "2025-01-10T10:00:00Z"
}
```

---

## 6. Backend Architecture

### 6.1 Service Layer

```
┌────────────────────────────────────────────────┐
│              TaskController                    │
│  (Thin layer: validation, routing)             │
└───────────────────────┬────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────┐
│              TaskService                       │
│  - Core CRUD logic                             │
│  - Optimistic locking enforcement              │
│  - Delegates to extension handlers             │
└───────────────────────┬────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│ NutritionalExt│ │ LeaveExtension│ │ (Future...)   │
│ Handler       │ │ Handler       │ │               │
└───────────────┘ └───────────────┘ └───────────────┘
```

### 6.2 Extension Handler Contract

```php
interface TaskExtensionHandler {
    public function validate(array $extensionData): void;
    public function create(Task $task, array $extensionData): void;
    public function update(Task $task, array $extensionData): void;
    public function delete(Task $task): void;
}
```

---

## 7. Migration Compatibility

During the transition period, the `tasks` table includes legacy compatibility fields:

| Field | Purpose |
| :--- | :--- |
| `legacy_merge_key` | Maps to the original `merge_key` for backfill and traceability. |
| `legacy_task_id` | Maps to the original `task.idtask` for audit. |
| `legacy_spraying_id` | Maps to the original `spraying.idspraying` for audit. |

These fields should be **removed** after the migration is complete and validated.

---



## 8. Design Decisions (ADRs)

### ADR-001: `idgroup` as Partition Key
**Decision:** Use `idgroup` (Operational Unit) instead of `idtenant` as the primary partition key.
**Rationale:** Work is scoped to a specific site. Multi-site tenants need data isolation per site.

### ADR-002: No `merge_key` in TO-BE
**Decision:** The `merge_key` concept is eliminated. The `tasks.id` is the canonical identifier.
**Rationale:** `merge_key` is a source of complexity and coupling. Proper FKs are superior.

### ADR-003: Optimistic Locking via `version`
**Decision:** Updates require the current `version` and increment it on success.
**Rationale:** Prevents lost updates in concurrent edit scenarios.

### ADR-004: Extension Tables for Domain-Specific Data
**Decision:** Use `task_ext_*` tables instead of a polymorphic `parameters` JSON blob for critical fields.
**Rationale:** Enables proper validation, indexing, and schema enforcement for compliance-critical data (e.g., Spraying).
