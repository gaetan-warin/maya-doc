# Fleet Management
## Technical Design Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-TSD-FLEET-001 |
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
│  Views                   │  Components             │  Store     │
│  ├─ fleet_management.vue │  ├─ FleetStatusOverview │  fleet.js  │
│  └─ machinery.vue        │  ├─ MaintenanceKanban   │            │
│                          │  ├─ FleetMachineList    │            │
│                          │  ├─ MachineryIndex      │            │
│                          │  └─ MachineryHours      │            │
├─────────────────────────────────────────────────────────────────┤
│                          API Layer                               │
│                    apiClient (Axios V2)                          │
├─────────────────────────────────────────────────────────────────┤
│                     Backend (Node.js)                            │
│              Fleet Service + Maintenance Service                 │
├─────────────────────────────────────────────────────────────────┤
│                      Database (MySQL)                            │
│    machine | maintenance | fuel_usage | machine_hours            │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Layer | Technology | Version |
|:---|:---|:---|
| Frontend Framework | Vue.js | 3.x |
| State Management | Pinia | 2.x |
| UI Components | CoreUI Vue | 4.x |
| Backend | Node.js / Express | 18.x |
| Database | MySQL | 8.x |

---

## 2. Component Architecture

### 2.1 View Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `fleet_management.vue` | `web/src/views/fleet_management.vue` | Main dashboard, Kanban/List toggle |
| `machinery.vue` | `web/src/views/machinery.vue` | Individual machine detail page |

### 2.2 Fleet Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `FleetStatusOverview.vue` | `web/src/components/fleet/` | Dashboard metrics display |
| `FleetStatusControl.vue` | `web/src/components/fleet/` | Filter controls |
| `MaintenanceKanban.vue` | `web/src/components/fleet/Maintenance Kanban/` | Kanban board view |
| `FleetMachineList.vue` | `web/src/components/fleet/machine-list/` | Table list view |
| `FleetIncident.vue` | `web/src/components/fleet/` | Incident logging |

### 2.3 Machinery Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `MachineryIndex.vue` | `web/src/components/fleet/machinery/` | Machine detail controller |
| `MachineryOverview.vue` | `web/src/components/fleet/machinery/` | Asset information display |
| `MachineryHours/` | `web/src/components/fleet/elements/` | Hours tracking components |
| `MachineryFuel.vue` | `web/src/components/fleet/elements/` | Fuel usage entry |
| `MachineryFiles.vue` | `web/src/components/fleet/elements/` | Document management |

---

## 3. State Management

### 3.1 Fleet Store

**File**: `web/src/store/fleet.js`

```javascript
// State Structure (partial)
{
  machineData: [],              // All machines
  maintenanceData: [],          // Maintenance records
  maintenanceUpcoming: [],      // Future maintenance
  maintenanceDue: [],           // Due maintenance
  urgentData: [],               // Urgent items
  minorData: [],                // Minor items
  fuelUsage: [],                // Fuel records
  machineTime: [],              // Hour logs
  selectedMachine: {},          // Current machine detail
  seletedMachineId: null,       // Current machine ID
  isLoading: false
}
```

### 3.2 Key Actions (79+ total)

| Action | Description | API Call |
|:---|:---|:---|
| `getAllMaintenanceData(filter, tags, query)` | Load maintenance | `GET /maintenance` |
| `getMachineData()` | Load all machines | `GET /machines` |
| `getMachineDataByMachine(id)` | Load single machine | `GET /machines/{id}` |
| `getFuelUsage(id)` | Load fuel records | `GET /fuel-usage/{id}` |
| `getMachineTime(id)` | Load hour logs | `GET /machine-time/{id}` |
| `updateMaintenanceStatus(id, status)` | Change status | `PATCH /maintenance/{id}` |
| `maintenanceStatusDone(id, data)` | Complete maintenance | `POST /maintenance/{id}/complete` |
| `addNewMachineIncident(body)` | Log incident | `POST /incidents` |
| `transferMachine(data)` | Transfer between tenants | `POST /transfer-machine` |

---

## 4. Data Models

### 4.1 Machine Entity

**Table**: `machine`

| Column | Type | Constraints | Description |
|:---|:---|:---|:---|
| `id` | CHAR(36) | PK | UUID identifier |
| `idtenant` | CHAR(36) | FK, NOT NULL | Tenant reference |
| `name` | VARCHAR(100) | NOT NULL | Machine name |
| `brand_id` | CHAR(36) | FK | Brand reference |
| `model_id` | CHAR(36) | FK | Model reference |
| `category_id` | CHAR(36) | FK | Category reference |
| `frame_number` | VARCHAR(50) | | Serial number |
| `fleet_number` | VARCHAR(20) | | Internal number |
| `purchase_date` | DATE | | Acquisition date |
| `purchase_price` | DECIMAL(12,2) | | Original cost |
| `ownership` | ENUM | | 'owned', 'leased', 'rented' |
| `status` | ENUM | | 'operational', 'service', 'out' |
| `total_hours` | DECIMAL(10,2) | | Lifetime hours |
| `created_at` | TIMESTAMP | | Creation timestamp |

### 4.2 Maintenance Entity

**Table**: `maintenance`

| Column | Type | Constraints | Description |
|:---|:---|:---|:---|
| `id` | CHAR(36) | PK | UUID identifier |
| `idmachine` | CHAR(36) | FK, NOT NULL | Machine reference |
| `idtenant` | CHAR(36) | FK | Tenant reference |
| `issue` | VARCHAR(255) | NOT NULL | Issue description |
| `priority` | ENUM | | 'urgent', 'minor', 'scheduled' |
| `status` | ENUM | | 'created', 'due', 'in_progress', 'completed' |
| `due_date` | DATE | | Target date |
| `due_hours` | DECIMAL(10,2) | | Hours threshold |
| `completed_date` | DATE | | Completion date |
| `cost` | DECIMAL(10,2) | | Total cost |
| `assigned_to` | CHAR(36) | FK | Technician reference |
| `notes` | TEXT | | Free-text notes |

### 4.3 Fuel Usage Entity

**Table**: `fuel_usage`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `idmachine` | CHAR(36) | Machine reference |
| `fuel_type_id` | CHAR(36) | Fuel type reference |
| `quantity` | DECIMAL(8,2) | Liters |
| `cost` | DECIMAL(10,2) | Fuel cost |
| `hours_at_fill` | DECIMAL(10,2) | Machine hours |
| `fill_date` | DATE | Fill date |

### 4.4 Machine Time Entity

**Table**: `machine_time`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `idmachine` | CHAR(36) | Machine reference |
| `hours` | DECIMAL(10,2) | Hours logged |
| `logged_date` | DATE | Entry date |
| `logged_by` | CHAR(36) | User reference |
| `source` | ENUM | 'manual', 'checklist', 'task' |

---

## 5. API Specification

### 5.1 Get All Maintenance

```
GET /api/v2/maintenance?idtenant={tenant}&status={status}
Authorization: Bearer {token}

Query Parameters:
- idtenant: UUID (required)
- idmachine: UUID (optional filter)
- status: 'urgent' | 'minor' | 'due' | 'upcoming' | 'history'
- tags: comma-separated tag IDs

Response: 200 OK
{
  "records": [
    {
      "id": "maint-uuid",
      "issue": "Oil Change",
      "priority": "scheduled",
      "status": "due",
      "machine": { "name": "Toro 3250", "total_hours": 1250 },
      "due_date": "2025-12-30"
    }
  ],
  "total": 15
}
```

### 5.2 Complete Maintenance

```
POST /api/v2/maintenance/{id}/complete
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "completed_date": "2025-12-30",
  "cost": 150.00,
  "labor_hours": 2.5,
  "parts_used": [
    { "product_id": "part-uuid", "quantity": 2 }
  ],
  "notes": "Replaced oil filter"
}

Response: 200 OK
{
  "id": "maint-uuid",
  "status": "completed",
  "message": "Maintenance completed successfully"
}
```

### 5.3 Log Machine Hours

```
POST /api/v2/machine-time
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "idmachine": "machine-uuid",
  "hours": 8.5,
  "logged_date": "2025-12-30",
  "source": "manual"
}

Response: 201 Created
{
  "id": "time-uuid",
  "new_total_hours": 1258.5
}
```

---

## 6. Sequence Diagrams

### 6.1 Maintenance Completion Flow

```
┌──────┐      ┌─────────────┐      ┌─────────┐      ┌─────────┐      ┌───────┐
│ User │      │ Kanban Card │      │  Store  │      │   API   │      │ Stock │
└──┬───┘      └──────┬──────┘      └────┬────┘      └────┬────┘      └───┬───┘
   │  Click Done     │                  │                │               │
   │────────────────>│                  │                │               │
   │                 │  Open Modal      │                │               │
   │                 │  Enter Details   │                │               │
   │  Submit         │                  │                │               │
   │────────────────>│                  │                │               │
   │                 │  maintenanceStatusDone()          │               │
   │                 │─────────────────>│                │               │
   │                 │                  │  POST complete │               │
   │                 │                  │───────────────>│               │
   │                 │                  │                │  Deduct Parts │
   │                 │                  │                │──────────────>│
   │                 │                  │  200 OK        │               │
   │                 │                  │<───────────────│               │
   │                 │  Move to History │                │               │
   │                 │<─────────────────│                │               │
```

---

## 7. Performance Requirements

| Metric | Requirement |
|:---|:---|
| Fleet dashboard load | < 3 seconds |
| Machine detail load | < 2 seconds |
| Maintenance status update | < 1 second |
| Hours log creation | < 500ms |

---

## 8. Dependencies

| Dependency | Type | Purpose |
|:---|:---|:---|
| Stock Module | Transactional | Parts consumption |
| MyChecklist | Integration | Pre-op inspection history |
| Task Module | Reference | Machine usage in tasks |
| Reporting | Consumer | Fleet cost reports |

---

**Document End**
