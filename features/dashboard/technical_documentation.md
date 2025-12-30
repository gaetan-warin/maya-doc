# Dashboard & Task Management
## Technical Design Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-TSD-DASH-001 |
| **Version** | 1.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Engineering Team |
| **Classification** | Internal - Technical |

---

## 1. System Architecture

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Frontend (Vue 3)                          │
├─────────────────────────────────────────────────────────────────┤
│  Views          │  Components       │  Store (Pinia)            │
│  ├─ schedule.vue│  ├─ TaskForm.vue  │  ├─ scheduleStore.js     │
│  └─ dayplan.vue │  ├─ StaffList.vue │  └─ all.js (shared)      │
│                 │  └─ TaskCard.vue  │                           │
├─────────────────────────────────────────────────────────────────┤
│                          API Layer                               │
│                    apiClient (Axios)                             │
├─────────────────────────────────────────────────────────────────┤
│                     Backend (Node.js)                            │
│                  REST API + OData endpoints                      │
├─────────────────────────────────────────────────────────────────┤
│                      Database (MySQL)                            │
│           schedule | task | staff | machinery                    │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Layer | Technology | Version |
|:---|:---|:---|
| Frontend Framework | Vue.js | 3.x |
| State Management | Pinia | 2.x |
| HTTP Client | Axios | 1.x |
| UI Components | CoreUI Vue | 4.x |
| Backend | Node.js / Express | 18.x |
| Database | MySQL | 8.x |
| ORM | Sequelize | 6.x |

---

## 2. Component Architecture

### 2.1 View Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `schedule.vue` | `web/src/views/schedule.vue` | Main scheduling view, task list display |
| `dayplan.vue` | `web/src/views/dayplan.vue` | Gantt-style daily planning interface |

### 2.2 Child Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `TaskForm.vue` | `web/src/components/schedule/TaskForm.vue` | Task creation/edit modal |
| `TaskCard.vue` | `web/src/components/schedule/TaskCard.vue` | Individual task display unit |
| `StaffSelector.vue` | `web/src/components/shared/StaffSelector.vue` | Multi-select staff picker |
| `MachinerySelector.vue` | `web/src/components/shared/MachinerySelector.vue` | Equipment assignment UI |

### 2.3 Composables

| Composable | Path | Purpose |
|:---|:---|:---|
| `useSchedule` | `web/src/composables/useSchedule.js` | Schedule state and actions |
| `useStaff` | `web/src/composables/useStaff.js` | Staff data management |
| `usePermissions` | `web/src/composables/usePermissions.js` | Role-based access control |

---

## 3. State Management

### 3.1 Schedule Store

**File**: `web/src/store/schedule.js`

```javascript
// State Structure
{
  tasks: [],              // Array of task objects
  selectedDate: null,     // Current view date
  filters: {
    staff: [],            // Staff filter array
    status: 'all',        // Status filter
    site: null            // Site filter
  },
  isLoading: false,
  error: null
}
```

### 3.2 Key Actions

| Action | Description | API Call |
|:---|:---|:---|
| `fetchTasks(date)` | Load tasks for date | `GET /api/v2/schedule` |
| `createTask(payload)` | Create new task | `POST /api/v2/createSchedule` |
| `updateTask(id, payload)` | Update existing task | `PUT /api/v2/schedule/{id}` |
| `deleteTask(id)` | Remove task | `DELETE /api/v2/schedule/{id}` |
| `updateStatus(id, status)` | Change task status | `PATCH /api/v2/schedule/{id}/status` |

---

## 4. Data Models

### 4.1 Task Entity

**Table**: `schedule`

| Column | Type | Constraints | Description |
|:---|:---|:---|:---|
| `id` | CHAR(36) | PK | UUID identifier |
| `idtenant` | CHAR(36) | FK, NOT NULL | Tenant reference |
| `idgroup` | CHAR(36) | FK | Group reference |
| `action_name` | VARCHAR(255) | NOT NULL | Task description |
| `start_date` | DATE | NOT NULL | Scheduled start |
| `end_date` | DATE | NOT NULL | Scheduled end |
| `planned_duration` | DECIMAL(6,2) | | Duration in hours |
| `running` | TINYINT | DEFAULT 0 | Completion flag |
| `task_start` | TIMESTAMP | NULL | Actual start time |
| `task_end` | TIMESTAMP | NULL | Actual end time |
| `is_overtime` | TINYINT | DEFAULT 0 | OT flag |
| `created_at` | TIMESTAMP | | Creation timestamp |
| `updated_at` | TIMESTAMP | | Last update |

### 4.2 Task Status Logic

| Status | `running` | `task_start` | `task_end` |
|:---|:---|:---|:---|
| `todo` | 0 | NULL | NULL |
| `in_progress` | 0 | value | NULL |
| `completed` | 1 | value | value |
| `not_completed` | 0 | value | value |

### 4.3 Related Tables

| Table | Relationship | Purpose |
|:---|:---|:---|
| `schedule_staff` | Many-to-Many | Staff assignments |
| `schedule_machinery` | Many-to-Many | Machine assignments |
| `schedule_sites` | Many-to-Many | Site/hole assignments |
| `schedule_tags` | Many-to-Many | Task categorization |

---

## 5. API Specification

### 5.1 Endpoints

#### Create Task
```
POST /api/v2/createSchedule
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "action_name": "Mow Greens",
  "start_date": "2025-12-30",
  "end_date": "2025-12-30",
  "planned_duration": 3.0,
  "staff_ids": ["uuid1", "uuid2"],
  "machinery_ids": ["uuid3"],
  "site_ids": ["uuid4"],
  "is_overtime": false
}

Response: 201 Created
{
  "id": "new-task-uuid",
  "message": "Task created successfully"
}
```

#### Get Tasks
```
GET /api/v2/schedule?date=2025-12-30&idsite=uuid
Authorization: Bearer {token}

Response: 200 OK
{
  "records": [
    {
      "id": "task-uuid",
      "action_name": "Mow Greens",
      "start_date": "2025-12-30",
      "staff": [...],
      "status": "todo"
    }
  ],
  "total": 15
}
```

### 5.2 Error Codes

| Code | Description | Resolution |
|:---|:---|:---|
| 400 | Invalid request body | Check required fields |
| 401 | Unauthorized | Refresh token |
| 403 | Permission denied | Check user role |
| 404 | Task not found | Verify task ID |
| 409 | Conflict (duplicate) | Check for existing |
| 500 | Server error | Contact support |

---

## 6. Sequence Diagrams

### 6.1 Task Creation Flow

```
┌──────┐          ┌──────────┐          ┌─────────┐          ┌────────┐
│ User │          │ TaskForm │          │  Store  │          │  API   │
└──┬───┘          └────┬─────┘          └────┬────┘          └───┬────┘
   │   Fill Form       │                     │                   │
   │──────────────────>│                     │                   │
   │                   │  Validate Inputs    │                   │
   │                   │─────────────────────│                   │
   │                   │  createTask()       │                   │
   │                   │────────────────────>│                   │
   │                   │                     │  POST /schedule   │
   │                   │                     │──────────────────>│
   │                   │                     │    201 Created    │
   │                   │                     │<──────────────────│
   │                   │  Update tasks[]     │                   │
   │                   │<────────────────────│                   │
   │   Close Modal     │                     │                   │
   │<──────────────────│                     │                   │
```

### 6.2 Mobile Status Update Flow

```
┌────────────┐       ┌─────────┐       ┌─────────┐       ┌──────────┐
│ Mobile App │       │   API   │       │   DB    │       │ Web View │
└─────┬──────┘       └────┬────┘       └────┬────┘       └────┬─────┘
      │  Start Task       │                 │                 │
      │──────────────────>│                 │                 │
      │                   │  UPDATE         │                 │
      │                   │────────────────>│                 │
      │                   │  Emit Event     │                 │
      │                   │─────────────────────────────────>│
      │                   │                 │   Refresh UI    │
      │   200 OK          │                 │                 │
      │<──────────────────│                 │                 │
```

---

## 7. Security Considerations

### 7.1 Authentication
- JWT Bearer tokens required for all API calls
- Token refresh handled automatically by Axios interceptor
- Session timeout: 24 hours

### 7.2 Authorization
| Permission | Description |
|:---|:---|
| `CREATE_SCHEDULE` | Create new tasks |
| `EDIT_SCHEDULE` | Modify existing tasks |
| `DELETE_SCHEDULE` | Remove tasks |
| `VIEW_ALL_STAFF` | See all staff assignments |

### 7.3 Data Isolation
- All queries filtered by `idtenant`
- Cross-tenant access prevented at API layer

---

## 8. Performance Requirements

| Metric | Requirement |
|:---|:---|
| Task list load time | < 2 seconds for 100 tasks |
| Create task response | < 1 second |
| Status update propagation | < 500ms to connected clients |
| Concurrent users | Support 50+ simultaneous |

---

## 9. Dependencies

| Dependency | Type | Purpose |
|:---|:---|:---|
| Weather Module | Data | Spray window recommendations |
| Stock Module | Transactional | Consumption recording |
| Mobile App | Integration | Real-time status updates |
| Reporting Module | Consumer | Labor analytics |

---

**Document End**
