# API Contract Specification
## Maya — Task & Scheduling API Reference (V3)

| Document Details | |
| :--- | :--- |
| **Project** | Maya Core 2.0 |
| **Module** | Scheduling & Task Management |
| **Version** | 3.0 |
| **Status** | Authoritative Reference |
| **Target Audience** | Frontend Developers, Backend Developers, QA |

---

## 1. Overview

This document specifies the API contracts for the Maya Scheduling module V3 architecture. This version eliminates the legacy `merge_key` pattern in favor of proper relational IDs and many-to-many junction tables.

### Design Principles

| Principle | Description |
| :--- | :--- |
| **One Task = One Row** | A single `tasks` table row represents one logical task. |
| **Backend as Source of Truth** | The backend generates and owns task identity (`tasks.id`). |
| **Many-to-Many Relations** | Staff, machines, locations, tags, and products use junction tables. |
| **Optimistic Locking** | Updates require version matching to prevent concurrent edit conflicts. |
| **Standardized Responses** | All endpoints return complete data with all relations included. |
| **Flat Relation Objects** | Relations use `{ id, name, ...custom_fields }` - no nested objects. |

### Base URLs

| Environment | URL |
| :--- | :--- |
| Development | `http://localhost:8000/api/v3` |
| Staging | `https://api-staging.maya.io/api/v3` |
| Production | `https://api.maya.io/api/v3` |

### Authentication

All endpoints require a valid Bearer token:

```
Authorization: Bearer <access_token>
```

---

## 2. Task Endpoints

### 2.1 List Tasks

Fetches tasks with optional filtering. All relations are always included.

**Endpoint:** `GET /tasks`

**Query Parameters:**

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `idgroup` | UUID | Yes | The Group (Operational Unit) ID |
| `start_date` | YYYY-MM-DD | No | Filter by planned_start >= |
| `end_date` | YYYY-MM-DD | No | Filter by planned_start <= |
| `type_id` | string | No | Filter by task type (`GENERAL`, `SPRAYING`, `LEAVE`) |
| `status` | string | No | Filter by status |
| `action_id` | UUID | No | Filter by action |
| `site_id` | UUID | No | Filter by site (via locations) |
| `user_id` | UUID | No | Filter by assigned staff (via assignments) |
| `page` | integer | No | Page number (default: 1) |
| `per_page` | integer | No | Items per page (default: 20, max: 100) |

**Example Request:**
```
GET /tasks?idgroup=550e8400-e29b-41d4-a716-446655440000&start_date=2025-12-15&end_date=2025-12-15
```

**Response (Success - 200):**
```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440099",
      "idgroup": "550e8400-e29b-41d4-a716-446655440000",
      "type_id": "GENERAL",
      "action_id": "550e8400-e29b-41d4-a716-446655440003",
      "action_name": "Mowing",
      "title": "Morning Mowing",
      "planned_start": "2025-12-15T08:00:00Z",
      "planned_end": "2025-12-15T11:00:00Z",
      "planned_minutes": 180,
      "actual_start": null,
      "actual_end": null,
      "status": "PLANNED",
      "version": 1,
      "notes": "Focus on greens",
      "assignments": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440020",
          "name": "John Doe",
          "role": "LEAD",
          "is_overtime": false
        },
        {
          "id": "550e8400-e29b-41d4-a716-446655440021",
          "name": "Jane Smith",
          "role": "WORKER",
          "is_overtime": false
        }
      ],
      "resources": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440030",
          "name": "Mower #1",
          "resource_type": "MACHINE"
        }
      ],
      "locations": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440002",
          "name": "North Course",
          "hole_id": null,
          "hole_name": null
        }
      ],
      "tags": [
        { "id": "550e8400-e29b-41d4-a716-446655440010", "name": "Greens" },
        { "id": "550e8400-e29b-41d4-a716-446655440011", "name": "Front 9" }
      ],
      "products": [],
      "created_at": "2025-12-10T10:00:00Z",
      "updated_at": "2025-12-10T10:00:00Z",
      "created_by": "550e8400-e29b-41d4-a716-446655440001"
    }
  ],
  "meta": {
    "current_page": 1,
    "last_page": 1,
    "per_page": 20,
    "total": 1
  }
}
```

---

### 2.2 Get Single Task

Fetches a single task with all relations.

**Endpoint:** `GET /tasks/{id}`

**Path Parameters:**

| Parameter | Type | Description |
| :--- | :--- | :--- |
| `id` | UUID | The task ID |

**Response (Success - 200):**
```json
{
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440099",
    "idgroup": "550e8400-e29b-41d4-a716-446655440000",
    "type_id": "GENERAL",
    "action_id": "550e8400-e29b-41d4-a716-446655440003",
    "action_name": "Mowing",
    "title": "Morning Mowing",
    "planned_start": "2025-12-15T08:00:00Z",
    "planned_end": "2025-12-15T11:00:00Z",
    "planned_minutes": 180,
    "actual_start": null,
    "actual_end": null,
    "status": "PLANNED",
    "version": 1,
    "notes": "Focus on greens",
    "assignments": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440020",
        "name": "John Doe",
        "role": "LEAD",
        "is_overtime": false,
        "planned_minutes": null,
        "actual_minutes": null
      }
    ],
    "resources": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440030",
        "name": "Mower #1",
        "resource_type": "MACHINE"
      }
    ],
    "locations": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440002",
        "name": "North Course",
        "hole_id": "550e8400-e29b-41d4-a716-446655440005",
        "hole_name": "Hole 1"
      }
    ],
    "tags": [
      { "id": "550e8400-e29b-41d4-a716-446655440010", "name": "Greens" }
    ],
    "products": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440070",
        "name": "Fertilizer X",
        "quantity": 10.5,
        "unit": "kg"
      }
    ],
    "created_at": "2025-12-10T10:00:00Z",
    "updated_at": "2025-12-10T10:00:00Z",
    "created_by": "550e8400-e29b-41d4-a716-446655440001"
  }
}
```

**Response (Not Found - 404):**
```json
{
  "error": "Not Found",
  "message": "Task not found"
}
```

---

### 2.3 Create Task

Creates a new task with all relations in a single atomic request.

**Endpoint:** `POST /tasks`

**Request Body:**
```json
{
  "idgroup": "550e8400-e29b-41d4-a716-446655440000",
  "type_id": "GENERAL",
  "action_id": "550e8400-e29b-41d4-a716-446655440003",
  "title": "Morning Mowing",
  "planned_start": "2025-12-15T08:00:00Z",
  "planned_end": "2025-12-15T11:00:00Z",
  "planned_minutes": 180,
  "notes": "Focus on greens",
  "locations": [
    { "id": "550e8400-e29b-41d4-a716-446655440002", "hole_id": null }
  ],
  "tags": [
    "550e8400-e29b-41d4-a716-446655440010",
    "550e8400-e29b-41d4-a716-446655440011"
  ],
  "assignments": [
    { "id": "550e8400-e29b-41d4-a716-446655440020", "role": "LEAD", "is_overtime": false },
    { "id": "550e8400-e29b-41d4-a716-446655440021", "role": "WORKER", "is_overtime": false }
  ],
  "resources": [
    { "id": "550e8400-e29b-41d4-a716-446655440030", "resource_type": "MACHINE" }
  ],
  "products": [
    { "id": "550e8400-e29b-41d4-a716-446655440070", "quantity": 10.5, "unit": "kg" }
  ]
}
```

**Required Fields:**

| Field | Type | Description |
| :--- | :--- | :--- |
| `idgroup` | UUID | The Group (Operational Unit) ID |
| `type_id` | string | Task type: `GENERAL`, `SPRAYING`, or `LEAVE` |
| `action_id` | UUID | The Action ID |
| `title` | string | Display name |
| `planned_start` | datetime | Planned start time (ISO 8601) |
| `assignments` | array | At least one staff assignment required |

**Optional Fields:**

| Field | Type | Description |
| :--- | :--- | :--- |
| `planned_end` | datetime | Planned end time (ISO 8601) |
| `planned_minutes` | integer | Estimated duration in minutes |
| `notes` | string | Additional notes |
| `locations` | array | Site/hole locations |
| `tags` | UUID[] | Array of tag IDs |
| `resources` | array | Machine/equipment assignments |
| `products` | array | Product allocations |
| *(spraying fields)* | various | See Section 4.2 for SPRAYING-specific fields |

**Response (Success - 201):**
```json
{
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440099",
    "idgroup": "550e8400-e29b-41d4-a716-446655440000",
    "type_id": "GENERAL",
    "action_id": "550e8400-e29b-41d4-a716-446655440003",
    "action_name": "Mowing",
    "title": "Morning Mowing",
    "planned_start": "2025-12-15T08:00:00Z",
    "planned_end": "2025-12-15T11:00:00Z",
    "planned_minutes": 180,
    "actual_start": null,
    "actual_end": null,
    "status": "PLANNED",
    "version": 1,
    "notes": "Focus on greens",
    "assignments": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440020",
        "name": "John Doe",
        "role": "LEAD",
        "is_overtime": false
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440021",
        "name": "Jane Smith",
        "role": "WORKER",
        "is_overtime": false
      }
    ],
    "resources": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440030",
        "name": "Mower #1",
        "resource_type": "MACHINE"
      }
    ],
    "locations": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440002",
        "name": "North Course",
        "hole_id": null,
        "hole_name": null
      }
    ],
    "tags": [
      { "id": "550e8400-e29b-41d4-a716-446655440010", "name": "Greens" },
      { "id": "550e8400-e29b-41d4-a716-446655440011", "name": "Front 9" }
    ],
    "products": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440070",
        "name": "Fertilizer X",
        "quantity": 10.5,
        "unit": "kg"
      }
    ],
    "created_at": "2025-12-10T10:00:00Z",
    "updated_at": "2025-12-10T10:00:00Z",
    "created_by": "550e8400-e29b-41d4-a716-446655440001"
  }
}
```

**Response (Validation Error - 422):**
```json
{
  "error": "Unprocessable Entity",
  "message": "Validation failed",
  "errors": {
    "assignments": ["At least one staff assignment is required"],
    "action_id": ["The action_id field is required"]
  }
}
```

---

### 2.4 Update Task

Updates an existing task with optimistic locking.

**Endpoint:** `PATCH /tasks/{id}`

**Path Parameters:**

| Parameter | Type | Description |
| :--- | :--- | :--- |
| `id` | UUID | The task ID |

**Request Body:**
```json
{
  "version": 1,
  "title": "Morning Mowing - Extended",
  "planned_end": "2025-12-15T12:00:00Z",
  "planned_minutes": 240,
  "status": "RUNNING",
  "actual_start": "2025-12-15T08:15:00Z",
  "notes": "Extended session due to weather delay",
  "assignments": [
    { "id": "550e8400-e29b-41d4-a716-446655440020", "role": "LEAD" },
    { "id": "550e8400-e29b-41d4-a716-446655440021", "role": "WORKER" },
    { "id": "550e8400-e29b-41d4-a716-446655440022", "role": "WORKER" }
  ],
  "resources": [
    { "id": "550e8400-e29b-41d4-a716-446655440030", "resource_type": "MACHINE" }
  ],
  "tags": [
    "550e8400-e29b-41d4-a716-446655440010"
  ]
}
```

**Optimistic Locking:**
- The `version` field is **required** for updates
- The request `version` must match the current database `version`
- On success, `version` is incremented by 1
- On mismatch, a `409 Conflict` is returned

**Partial Updates:**
- Only include fields you want to change
- Relations (assignments, resources, locations, tags, products) are **replaced** entirely when provided
- To add/remove a single assignment, include the full desired list

**Response (Success - 200):**
```json
{
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440099",
    "idgroup": "550e8400-e29b-41d4-a716-446655440000",
    "type_id": "GENERAL",
    "action_id": "550e8400-e29b-41d4-a716-446655440003",
    "action_name": "Mowing",
    "title": "Morning Mowing - Extended",
    "planned_start": "2025-12-15T08:00:00Z",
    "planned_end": "2025-12-15T12:00:00Z",
    "planned_minutes": 240,
    "actual_start": "2025-12-15T08:15:00Z",
    "actual_end": null,
    "status": "RUNNING",
    "version": 2,
    "notes": "Extended session due to weather delay",
    "assignments": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440020",
        "name": "John Doe",
        "role": "LEAD",
        "is_overtime": false
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440021",
        "name": "Jane Smith",
        "role": "WORKER",
        "is_overtime": false
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440022",
        "name": "Bob Wilson",
        "role": "WORKER",
        "is_overtime": false
      }
    ],
    "resources": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440030",
        "name": "Mower #1",
        "resource_type": "MACHINE"
      }
    ],
    "locations": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440002",
        "name": "North Course",
        "hole_id": null,
        "hole_name": null
      }
    ],
    "tags": [
      { "id": "550e8400-e29b-41d4-a716-446655440010", "name": "Greens" }
    ],
    "products": [],
    "created_at": "2025-12-10T10:00:00Z",
    "updated_at": "2025-12-15T08:16:00Z",
    "created_by": "550e8400-e29b-41d4-a716-446655440001"
  }
}
```

**Response (Conflict - 409):**
```json
{
  "error": "Conflict",
  "message": "The task has been modified by another user. Please refresh and try again.",
  "current_version": 2
}
```

---

### 2.5 Delete Task

Deletes a task and all its relations (assignments, resources, locations, tags, products).

**Endpoint:** `DELETE /tasks/{id}`

**Path Parameters:**

| Parameter | Type | Description |
| :--- | :--- | :--- |
| `id` | UUID | The task ID |

**Response (Success - 204):**
```
(No Content)
```

**Response (Not Found - 404):**
```json
{
  "error": "Not Found",
  "message": "Task not found"
}
```

---

## 3. Task Status Transitions

### 3.1 Status Values

| Status | Description |
| :--- | :--- |
| `DRAFT` | Task is being created, not yet scheduled |
| `PLANNED` | Task is scheduled but not started |
| `RUNNING` | Task is currently in progress |
| `COMPLETED` | Task has been finished |
| `CANCELLED` | Task was cancelled |

### 3.2 Valid Transitions

```
DRAFT ──────► PLANNED ──────► RUNNING ──────► COMPLETED
                │                │
                │                └──────────► CANCELLED
                │
                └──────────────────────────► CANCELLED
```

### 3.3 Status Change Side Effects

| Transition | Side Effect |
| :--- | :--- |
| `PLANNED` → `RUNNING` | Sets `actual_start` if not provided; triggers inventory deduction for products |
| `RUNNING` → `COMPLETED` | Sets `actual_end` if not provided; finalizes inventory records |
| `RUNNING` → `CANCELLED` | Refunds inventory deductions |
| `PLANNED` → `CANCELLED` | No inventory changes |

---

## 4. Spraying Tasks (Type-Specific Fields)

For tasks with `type_id: "SPRAYING"`, additional fields are included directly in the request/response. The backend determines which fields to include based on `type_id`.

### 4.1 Create Spraying Task

**Request Body:**
```json
{
  "idgroup": "550e8400-e29b-41d4-a716-446655440000",
  "type_id": "SPRAYING",
  "action_id": "550e8400-e29b-41d4-a716-446655440003",
  "title": "Fungicide Application",
  "planned_start": "2025-12-15T06:00:00Z",
  "planned_end": "2025-12-15T08:00:00Z",
  "locations": [
    { "id": "550e8400-e29b-41d4-a716-446655440002" }
  ],
  "assignments": [
    { "id": "550e8400-e29b-41d4-a716-446655440020", "role": "OPERATOR" }
  ],
  "resources": [
    { "id": "550e8400-e29b-41d4-a716-446655440040", "resource_type": "SPRAYER" }
  ],
  "products": [
    { "id": "550e8400-e29b-41d4-a716-446655440075", "quantity": 25.0, "unit": "L" }
  ],
  "nozzle_type_id": "550e8400-e29b-41d4-a716-446655440060",
  "spacing_id": "550e8400-e29b-41d4-a716-446655440061",
  "speed": 8.5,
  "rpm": 540,
  "pressure": 2.5,
  "temperature": 18.5,
  "wind_speed": 5.2,
  "ppe_worn": true,
  "exclusion_zone_marked": true,
  "reentry_interval_hours": 24
}
```

### 4.2 Spraying-Specific Fields

These fields are only present when `type_id: "SPRAYING"`:

| Field | Type | Description |
| :--- | :--- | :--- |
| `nozzle_type_id` | UUID | Reference to nozzle type (request) |
| `nozzle_type_name` | string | Nozzle type name (response only) |
| `spacing_id` | UUID | Reference to spacing configuration (request) |
| `spacing_name` | string | Spacing name (response only) |
| `speed` | decimal | Application speed (km/h) |
| `rpm` | integer | Engine/PTO RPM |
| `pressure` | decimal | Spray pressure (bar) |
| `temperature` | decimal | Ambient temperature (°C) |
| `wind_speed` | decimal | Wind speed (km/h) |
| `ppe_worn` | boolean | PPE compliance flag |
| `exclusion_zone_marked` | boolean | Exclusion zone compliance flag |
| `reentry_interval_hours` | integer | Reentry interval in hours |

### 4.3 Spraying Task Response

```json
{
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440100",
    "idgroup": "550e8400-e29b-41d4-a716-446655440000",
    "type_id": "SPRAYING",
    "action_id": "550e8400-e29b-41d4-a716-446655440004",
    "action_name": "Fungicide Application",
    "title": "Greens Treatment",
    "planned_start": "2025-12-15T06:00:00Z",
    "planned_end": "2025-12-15T08:00:00Z",
    "status": "PLANNED",
    "version": 1,
    "assignments": [],
    "resources": [],
    "locations": [],
    "tags": [],
    "products": [],
    "nozzle_type_id": "550e8400-e29b-41d4-a716-446655440060",
    "nozzle_type_name": "Flat Fan 110°",
    "spacing_id": "550e8400-e29b-41d4-a716-446655440061",
    "spacing_name": "50cm",
    "speed": 8.5,
    "rpm": 540,
    "pressure": 2.5,
    "temperature": 18.5,
    "wind_speed": 5.2,
    "ppe_worn": true,
    "exclusion_zone_marked": true,
    "reentry_interval_hours": 24,
    "created_at": "2025-12-10T10:00:00Z",
    "updated_at": "2025-12-10T10:00:00Z"
  }
}
```

---

## 5. Calendar View Endpoint

Optimized endpoint for calendar display with summarized data.

### 5.1 Get Calendar Data

**Endpoint:** `GET /calendar`

**Query Parameters:**

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `idgroup` | UUID | Yes | The Group (Operational Unit) ID |
| `start_date` | YYYY-MM-DD | Yes | Start of date range |
| `end_date` | YYYY-MM-DD | Yes | End of date range |

**Response (Success - 200):**
```json
{
  "data": {
    "2025-12-15": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440099",
        "type_id": "GENERAL",
        "action_id": "550e8400-e29b-41d4-a716-446655440003",
        "action_name": "Mowing",
        "title": "Morning Mowing",
        "planned_start": "2025-12-15T08:00:00Z",
        "planned_end": "2025-12-15T11:00:00Z",
        "status": "PLANNED",
        "assignments": [
          { "id": "550e8400-e29b-41d4-a716-446655440020", "name": "John Doe", "role": "LEAD" },
          { "id": "550e8400-e29b-41d4-a716-446655440021", "name": "Jane Smith", "role": "WORKER" }
        ],
        "locations": [
          { "id": "550e8400-e29b-41d4-a716-446655440002", "name": "North Course" }
        ]
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440100",
        "type_id": "SPRAYING",
        "action_id": "550e8400-e29b-41d4-a716-446655440004",
        "action_name": "Fungicide Application",
        "title": "Greens Treatment",
        "planned_start": "2025-12-15T06:00:00Z",
        "planned_end": "2025-12-15T08:00:00Z",
        "status": "COMPLETED",
        "assignments": [
          { "id": "550e8400-e29b-41d4-a716-446655440023", "name": "Mike Johnson", "role": "OPERATOR" }
        ],
        "locations": [
          { "id": "550e8400-e29b-41d4-a716-446655440003", "name": "South Course" }
        ],
        "products": [
          { "id": "550e8400-e29b-41d4-a716-446655440075", "name": "Fungicide Pro", "quantity": 25.0, "unit": "L" }
        ]
      }
    ],
    "2025-12-16": []
  }
}
```

---

## 6. Bulk Operations

### 6.1 Bulk Create Tasks

Creates multiple tasks in a single request.

**Endpoint:** `POST /tasks/bulk`

**Request Body:**
```json
{
  "tasks": [
    {
      "idgroup": "550e8400-e29b-41d4-a716-446655440000",
      "type_id": "GENERAL",
      "action_id": "550e8400-e29b-41d4-a716-446655440003",
      "title": "Morning Mowing - Hole 1",
      "planned_start": "2025-12-15T08:00:00Z",
      "assignments": [{ "id": "550e8400-e29b-41d4-a716-446655440020", "role": "WORKER" }]
    },
    {
      "idgroup": "550e8400-e29b-41d4-a716-446655440000",
      "type_id": "GENERAL",
      "action_id": "550e8400-e29b-41d4-a716-446655440003",
      "title": "Morning Mowing - Hole 2",
      "planned_start": "2025-12-15T08:00:00Z",
      "assignments": [{ "id": "550e8400-e29b-41d4-a716-446655440021", "role": "WORKER" }]
    }
  ]
}
```

**Response (Success - 201):**
```json
{
  "data": {
    "created": 2,
    "tasks": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440101",
        "title": "Morning Mowing - Hole 1",
        "status": "PLANNED",
        "version": 1
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440102",
        "title": "Morning Mowing - Hole 2",
        "status": "PLANNED",
        "version": 1
      }
    ]
  }
}
```

### 6.2 Bulk Update Tasks

Updates multiple tasks in a single request.

**Endpoint:** `PATCH /tasks/bulk`

**Request Body:**
```json
{
  "updates": [
    { "id": "550e8400-e29b-41d4-a716-446655440099", "version": 1, "status": "RUNNING" },
    { "id": "550e8400-e29b-41d4-a716-446655440100", "version": 1, "status": "RUNNING" }
  ]
}
```

**Response (Success - 200):**
```json
{
  "data": {
    "updated": 2,
    "failed": 0,
    "results": [
      { "id": "550e8400-e29b-41d4-a716-446655440099", "success": true, "version": 2 },
      { "id": "550e8400-e29b-41d4-a716-446655440100", "success": true, "version": 2 }
    ]
  }
}
```

### 6.3 Bulk Delete Tasks

Deletes multiple tasks in a single request.

**Endpoint:** `DELETE /tasks/bulk`

**Request Body:**
```json
{
  "ids": [
    "550e8400-e29b-41d4-a716-446655440099",
    "550e8400-e29b-41d4-a716-446655440100"
  ]
}
```

**Response (Success - 200):**
```json
{
  "data": {
    "deleted": 2
  }
}
```

---

## 7. Standard Response Structure

All endpoints follow a consistent response structure.

### 7.1 Relation Object Pattern

All many-to-many relations follow the same flat pattern:

```json
{
  "id": "UUID",           // The referenced entity's ID
  "name": "string",       // The referenced entity's name
  "custom_field": "..."   // Any junction-specific fields
}
```

### 7.2 Relation Schemas

**Assignment:**
```json
{
  "id": "UUID",              // User ID
  "name": "string",          // User name
  "role": "LEAD | WORKER | OPERATOR | SUPPORT",
  "is_overtime": "boolean",
  "planned_minutes": "integer | null",
  "actual_minutes": "integer | null"
}
```

**Resource:**
```json
{
  "id": "UUID",              // Resource/Machine ID
  "name": "string",          // Resource name
  "resource_type": "MACHINE | IMPLEMENT | VEHICLE | SPRAYER"
}
```

**Location:**
```json
{
  "id": "UUID",              // Site ID
  "name": "string",          // Site name
  "hole_id": "UUID | null",  // Optional hole ID
  "hole_name": "string | null"  // Optional hole name
}
```

**Tag:**
```json
{
  "id": "UUID",              // Tag ID
  "name": "string"           // Tag name
}
```

**Product:**
```json
{
  "id": "UUID",              // Product ID
  "name": "string",          // Product name
  "quantity": "decimal",
  "unit": "string | null"
}
```

### 7.3 Empty Relations

Relations that have no data are returned as empty arrays, never `null`:

```json
{
  "assignments": [],
  "resources": [],
  "locations": [],
  "tags": [],
  "products": []
}
```

---

## 8. Error Codes Reference

| HTTP Code | Error | Description |
| :--- | :--- | :--- |
| `200` | Success | Request completed successfully |
| `201` | Created | Resource created successfully |
| `204` | No Content | Resource deleted successfully |
| `400` | Bad Request | Malformed JSON or invalid parameters |
| `401` | Unauthorized | Missing or invalid Bearer token |
| `403` | Forbidden | User lacks permission for this resource |
| `404` | Not Found | Resource does not exist |
| `409` | Conflict | Optimistic locking conflict (version mismatch) |
| `422` | Unprocessable Entity | Validation failed |
| `500` | Internal Server Error | Unexpected server error |

---

## 9. Data Types Reference

| Type | Format | Example |
| :--- | :--- | :--- |
| UUID | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` | `550e8400-e29b-41d4-a716-446655440000` |
| Datetime | ISO 8601 | `2025-12-15T08:00:00Z` |
| Date | `YYYY-MM-DD` | `2025-12-15` |
| Status | Enum | `DRAFT`, `PLANNED`, `RUNNING`, `COMPLETED`, `CANCELLED` |
| Type ID | String | `GENERAL`, `SPRAYING`, `LEAVE` |
| Role | String | `LEAD`, `WORKER`, `OPERATOR`, `SUPPORT` |
| Resource Type | String | `MACHINE`, `IMPLEMENT`, `VEHICLE`, `SPRAYER` |

---

## 10. Migration Notes

This V3 API replaces the legacy V2 API which used `merge_key` for task grouping. Key differences:

| Aspect | Legacy (V2) | Current (V3) |
| :--- | :--- | :--- |
| Task Identity | `merge_key` (frontend-generated string) | `tasks.id` (backend-generated UUID) |
| Staff Assignment | Child rows in `task` table | `task_assignments` junction table |
| Machine Assignment | Child rows in `task` table | `task_resources` junction table |
| Product Linking | `product_task` keyed by `merge_key` hash | `task_products` keyed by `task_id` |
| Concurrent Edits | No protection | Optimistic locking via `version` |
| Task Types | Separate `spraying` table | Unified `tasks` table with `type_id` discriminator |
| Response Data | Optional `include` parameter | All relations always included |
| Relation Format | Nested objects with redundant IDs | Flat `{ id, name, ...fields }` |

For migration details, see [04_MigrationPlaybook.md](04_MigrationPlaybook.md).
