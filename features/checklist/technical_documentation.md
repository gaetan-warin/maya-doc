# MyChecklist
## Technical Design Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-TSD-CHECK-001 |
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
│  Views                │  Components              │  Store       │
│  └─ my_checkLists.vue │  ├─ MyCheckList         │  my-check-  │
│                       │  ├─ CheckListManager     │  lists/      │
│                       │  ├─ CheckListHistory     │  index.js    │
│                       │  └─ CheckListForm        │              │
├─────────────────────────────────────────────────────────────────┤
│                          API Layer                               │
│                    apiClient (Axios V2)                          │
├─────────────────────────────────────────────────────────────────┤
│                     Backend (Node.js)                            │
│                Checklist Service + History Service               │
├─────────────────────────────────────────────────────────────────┤
│                      Database (MySQL)                            │
│  my_checklist | checklist_item | checklist_assignment | history │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Layer | Technology | Version |
|:---|:---|:---|
| Frontend Framework | Vue.js | 3.x (Composition API) |
| State Management | Pinia | 2.x |
| Backend | Node.js / Express | 18.x |
| Database | MySQL | 8.x |

---

## 2. Component Architecture

### 2.1 View Component

| Component | Path | Responsibility |
|:---|:---|:---|
| `my_checkLists.vue` | `web/src/views/my_checkLists.vue` | Tab container, navigation |

### 2.2 Tab Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `my-checklists/index.vue` | `web/src/components/check-lists/` | Template list display |
| `checklist-manager/index.vue` | `web/src/components/check-lists/` | Machine assignment grid |
| `checklist-history/index.vue` | `web/src/components/check-lists/` | History records |
| `my-checklists/form.vue` | `web/src/components/check-lists/` | Template create/edit modal |

### 2.3 Supporting Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `change-assign-checklist.vue` | `web/src/components/check-lists/checklist-manager/` | Assignment modal |
| `single-checklist.vue` | `web/src/components/check-lists/checklist-history/` | History detail modal |

---

## 3. State Management

### 3.1 Checklists Store

**File**: `web/src/store/my-checklists/index.js`

```javascript
// State Structure
{
  isMyChecklistsOpen: false,   // Form modal visibility
  data: [],                    // Checklist templates
  total: 0,                    // Pagination total
  checkListObj: {},            // Current template being edited
  checkLists: [],              // All checklists for assignment
  checkListMgtData: [],        // Machines with assignments
  checkListMgtTotal: 0,        // Manager pagination
  checkListHistoryData: [],    // History records
  checkListHistoryTotal: 0,    // History pagination
  singleCheckListHistoryData: [],  // Detail view data
  searchKey: '',               // Manager search filter
  isLoading: false,
  isCheckListSaving: false
}
```

### 3.2 Key Actions

| Action | Description | API Call |
|:---|:---|:---|
| `saveMyCheckLists(data)` | Create/update template | `POST /my-checklist` |
| `getCheckLists(page)` | Load templates | `GET /my-checklist` |
| `getCkechListById(id)` | Load single template | `GET /my-checklist/{id}` |
| `checkListDelete()` | Delete template | `DELETE /my-checklist/{id}` |
| `assignCheckListForMachine(data)` | Assign to machine | `POST /assign-checklist-for-machine` |
| `getMyCheckForMachine(id)` | Get machine assignments | `GET /get-check-lists-by-machine/{id}` |
| `getMachineListWithCheckLists(page)` | Load manager view | `GET /machine-detail-with-check-lists` |
| `getCheckListsHisoty(page, sort, order)` | Load history | `GET /checklist-history` |
| `getSingleCheckListHistory(id)` | Load history detail | `GET /checklist-submit-history/{id}` |
| `changeCheckListItem(id)` | Toggle item status | `GET /change-checklist-item-status/{id}` |

---

## 4. Data Models

### 4.1 Checklist Template Entity

**Table**: `my_checklist`

| Column | Type | Constraints | Description |
|:---|:---|:---|:---|
| `id` | CHAR(36) | PK | UUID identifier |
| `idtenant` | CHAR(36) | FK, NOT NULL | Tenant reference |
| `name` | VARCHAR(100) | NOT NULL | Checklist name |
| `is_active` | TINYINT | DEFAULT 1 | Active flag |
| `require_hours` | TINYINT | DEFAULT 0 | Require hour entry |
| `created_at` | TIMESTAMP | | Creation timestamp |
| `updated_at` | TIMESTAMP | | Last update |

### 4.2 Checklist Item Entity

**Table**: `checklist_item`

| Column | Type | Constraints | Description |
|:---|:---|:---|:---|
| `id` | CHAR(36) | PK | UUID identifier |
| `checklist_id` | CHAR(36) | FK, NOT NULL | Parent checklist |
| `description` | VARCHAR(255) | NOT NULL | Action description |
| `order` | INT | | Display sequence |
| `is_required` | TINYINT | DEFAULT 1 | Required flag |

### 4.3 Machine Assignment Entity

**Table**: `checklist_machine_assignment`

| Column | Type | Constraints | Description |
|:---|:---|:---|:---|
| `id` | CHAR(36) | PK | UUID identifier |
| `checklist_id` | CHAR(36) | FK, NOT NULL | Checklist reference |
| `machine_id` | CHAR(36) | FK, NOT NULL | Machine reference |
| `assigned_by` | CHAR(36) | FK | User who assigned |
| `assigned_at` | TIMESTAMP | | Assignment timestamp |

### 4.4 Checklist History Entity

**Table**: `checklist_history`

| Column | Type | Constraints | Description |
|:---|:---|:---|:---|
| `id` | CHAR(36) | PK | UUID identifier |
| `checklist_id` | CHAR(36) | FK | Template reference |
| `machine_id` | CHAR(36) | FK | Machine reference |
| `completed_by` | CHAR(36) | FK | User who completed |
| `completed_at` | TIMESTAMP | | Submission timestamp |
| `hours_logged` | DECIMAL(10,2) | | Hours reading |

### 4.5 History Item Entity

**Table**: `checklist_history_item`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `history_id` | CHAR(36) | Parent history record |
| `item_id` | CHAR(36) | Original checklist item |
| `status` | ENUM | 'passed', 'failed', 'skipped' |
| `notes` | TEXT | Optional notes |

---

## 5. API Specification

### 5.1 Get Checklist Templates

```
GET /api/v2/my-checklist?idtenant={tenant}&page={page}
Authorization: Bearer {token}

Response: 200 OK
{
  "data": {
    "data": [
      {
        "id": "checklist-uuid",
        "name": "Daily Mower Check",
        "require_hours": true,
        "items_count": 6
      }
    ],
    "total": 5
  }
}
```

### 5.2 Create Checklist Template

```
POST /api/v2/my-checklist?idtenant={tenant}
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "name": "Weekly Tractor Inspection",
  "require_hours": true,
  "items": [
    { "description": "Check oil level", "is_required": true, "order": 1 },
    { "description": "Inspect tires", "is_required": true, "order": 2 },
    { "description": "Test brakes", "is_required": true, "order": 3 }
  ]
}

Response: 201 Created
{
  "data": {
    "id": "new-checklist-uuid"
  }
}
```

### 5.3 Assign Checklist to Machine

```
POST /api/v2/assign-checklist-for-machine?idtenant={tenant}
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "machine_id": "machine-uuid",
  "checklist_ids": ["checklist-uuid-1", "checklist-uuid-2"]
}

Response: 200 OK
{
  "message": "Checklists assigned successfully"
}
```

### 5.4 Get Checklist History

```
GET /api/v2/checklist-history?idtenant={tenant}&page={page}&sortKey=completed_at&sortOrder=desc
Authorization: Bearer {token}

Response: 200 OK
{
  "data": {
    "data": [
      {
        "id": "history-uuid",
        "checklist_name": "Daily Mower Check",
        "machine_name": "Toro 3250",
        "completed_by_name": "John Smith",
        "completed_at": "2025-12-30T08:00:00Z",
        "hours_logged": 1258.5
      }
    ],
    "total": 100
  }
}
```

---

## 6. Sequence Diagrams

### 6.1 Checklist Submission Flow

```
┌──────────┐    ┌────────────┐    ┌─────────┐    ┌─────────┐    ┌───────────┐
│ Operator │    │ Mobile App │    │   API   │    │   DB    │    │ Fleet Svc │
└────┬─────┘    └─────┬──────┘    └────┬────┘    └────┬────┘    └─────┬─────┘
     │  Complete Items │               │              │               │
     │────────────────>│               │              │               │
     │  Enter Hours    │               │              │               │
     │────────────────>│               │              │               │
     │  Submit         │               │              │               │
     │────────────────>│               │              │               │
     │                 │  POST submit  │              │               │
     │                 │──────────────>│              │               │
     │                 │               │ INSERT hist  │               │
     │                 │               │─────────────>│               │
     │                 │               │ Update hours │               │
     │                 │               │─────────────────────────────>│
     │                 │  200 OK       │              │               │
     │                 │<──────────────│              │               │
     │  Confirmation   │               │              │               │
     │<────────────────│               │              │               │
```

---

## 7. Integration Points

| System | Type | Data Flow |
|:---|:---|:---|
| Fleet Management | Bidirectional | Machine assignment, hours update |
| Mobile App | Execution | Checklist completion |
| Reporting | Consumer | Compliance reports |

---

## 8. Performance Requirements

| Metric | Requirement |
|:---|:---|
| Template list load | < 2 seconds |
| Manager view load | < 3 seconds |
| History load | < 2 seconds |
| Submission processing | < 1 second |

---

**Document End**
