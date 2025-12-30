# Team Management
## Technical Design Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-TSD-TEAM-001 |
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
│  Views              │  Components              │  Stores         │
│  ├─ my-team.vue     │  ├─ StaffTableHeatMap   │  team-mgt.js    │
│  ├─ team.vue        │  ├─ LeaveRequest        │  permission/    │
│  └─ user.vue        │  ├─ TeamForm            │    ├─ store.ts  │
│                     │  ├─ UserForm            │    ├─ logic.ts  │
│                     │  └─ RoleForm            │    └─ api.ts    │
├─────────────────────────────────────────────────────────────────┤
│                          API Layer                               │
│                    apiClient (Axios V2)                          │
├─────────────────────────────────────────────────────────────────┤
│                     Backend (Node.js)                            │
│      UserService + TeamService + PermissionService + LeaveService│
├─────────────────────────────────────────────────────────────────┤
│                      Database (MySQL)                            │
│         user | team | permission_role | leave | toil            │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Layer | Technology | Version |
|:---|:---|:---|
| Frontend Framework | Vue.js | 3.x |
| State Management | Pinia | 2.x |
| Permission System | TypeScript | Custom store |
| Authentication | JWT | - |
| Backend | Node.js / Express | 18.x |

---

## 2. Component Architecture

### 2.1 View Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `my-team.vue` | `views/` | Staff availability, leave (730 lines) |
| `team.vue` | `views/` | Team CRUD (301 lines) |
| `user.vue` | `views/` | User management (446 lines) |

### 2.2 My Team Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `StaffTableHeatMap.vue` | `components/myTeam/` | Visual calendar grid |
| `TimelineCalendar.vue` | `components/myTeam/` | Timeline view |
| `LeaveRequest.vue` | `components/myTeam/` | Leave request list |
| `DayoffForm.vue` | `components/myTeam/` | Leave creation modal |
| `StaffProfile.vue` | `components/myTeam/` | Staff detail sidebar |
| `LeaveUpdateForm.vue` | `components/myTeam/` | Leave edit modal |

### 2.3 Team/User Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `TeamForm.vue` | `components/team/` | Team create/edit |
| `TeamTable.vue` | `components/team/` | Team list |
| `UserForm.vue` | `components/user/` | User create/edit |
| `UserTable.vue` | `components/user/` | User list |
| `RoleForm.vue` | `components/user/` | Permission role editor |
| `Lists.vue` | `components/user/` | Additional roles list |

---

## 3. State Management

### 3.1 Team Management Store

**File**: `web/src/store/team-mgt.js`

```javascript
// State Structure
{
  userInfo: JSON.parse(localStorage.getItem('userInfo')) || {},
  api: piniaEndPoint()
}

// Actions
getTeamMgtData() → GET /get-leaves-working-days/{tenantId}
```

### 3.2 Permission Store

**Directory**: `web/src/store/permission/`

| File | Purpose |
|:---|:---|
| `index.ts` | Store export |
| `store.ts` | Pinia store definition |
| `api.ts` | Permission API calls |
| `logic.ts` | Permission evaluation logic |
| `computed.ts` | Computed permission checks |
| `constants.ts` | Permission constants (10KB) |
| `types.ts` | TypeScript interfaces |
| `utils.ts` | Helper functions |

### 3.3 Permission Constants

```typescript
// constants.ts - Key exports
export const ENTITIES = {
  INVENTORY: 'inventory',
  FLEET: 'fleet',
  NUTRITIONAL: 'nutritional',
  WATER: 'water',
  REPORTS: 'reports',
  AGRONOMY: 'agronomy',
  PLANNER: 'planner',
  SCHEDULE: 'schedule'
}

export const CRUD_ACTIONS = {
  CREATE: 'create',
  READ: 'read',
  UPDATE: 'update',
  DELETE: 'delete'
}

export const PERMISSIONS = {
  INVENTORY: { MANAGE_STOCK: 'manage_stock', ... },
  FLEET: { MANAGE_MAINTENANCE: 'manage_maintenance', ... },
  ...
}
```

---

## 4. Data Models

### 4.1 User Entity

**Table**: `user`

| Column | Type | Constraints | Description |
|:---|:---|:---|:---|
| `iduser` | CHAR(36) | PK | UUID identifier |
| `idtenant` | CHAR(36) | FK | Tenant reference |
| `email` | VARCHAR(255) | UNIQUE | Login email |
| `password_hash` | VARCHAR(255) | | Hashed password |
| `first_name` | VARCHAR(100) | NOT NULL | Given name |
| `last_name` | VARCHAR(100) | NOT NULL | Family name |
| `role` | ENUM | | 'admin', 'viewer' |
| `idpermission_role` | CHAR(36) | FK | Custom role reference |
| `is_staff` | TINYINT | | Scheduling visibility |
| `mobile_access` | TINYINT | | App access flag |
| `idteam` | CHAR(36) | FK | Team assignment |
| `is_team_manager` | TINYINT | | Manager flag |
| `created_at` | TIMESTAMP | | Creation timestamp |

### 4.2 Team Entity

**Table**: `team`

| Column | Type | Description |
|:---|:---|:---|
| `idteam` | CHAR(36) | UUID identifier |
| `idtenant` | CHAR(36) | Tenant reference |
| `name` | VARCHAR(100) | Team name |
| `default_days_off` | INT | Annual leave allowance |
| `work_hours_per_day` | DECIMAL(4,2) | Standard hours |
| `site_ids` | JSON | Associated sites |

### 4.3 Permission Role Entity

**Table**: `permission_role`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `idtenant` | CHAR(36) | Tenant reference |
| `name` | VARCHAR(100) | Role name |
| `permissions` | JSON | Permission configuration |

### 4.4 Leave Entity

**Table**: `leave`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `iduser` | CHAR(36) | Staff reference |
| `leave_type` | ENUM | 'annual', 'sick', 'absent' |
| `start_date` | DATE | Leave start |
| `end_date` | DATE | Leave end |
| `status` | ENUM | 'pending', 'approved', 'rejected' |
| `reason` | TEXT | Leave reason |
| `comment` | TEXT | Additional notes |
| `approved_by` | CHAR(36) | Approver reference |

### 4.5 TOIL Entity

**Table**: `toil`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `iduser` | CHAR(36) | Staff reference |
| `date` | DATE | TOIL date |
| `hours` | DECIMAL(4,2) | Hours accumulated |
| `type` | ENUM | 'overtime', 'used' |
| `notes` | TEXT | Description |

---

## 5. API Specification

### 5.1 Get Users

```
GET /api/v2/getUsers?idtenant={tenant}
Authorization: Bearer {token}

Response: 200 OK
{
  "records": [
    {
      "iduser": "user-uuid",
      "first_name": "John",
      "last_name": "Smith",
      "email": "john@example.com",
      "role": "admin",
      "is_staff": true,
      "team_name": "Grounds Team"
    }
  ]
}
```

### 5.2 Create User

```
POST /api/v2/users
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "first_name": "John",
  "last_name": "Smith",
  "email": "john@example.com",
  "password": "***",
  "role": "viewer",
  "idpermission_role": "role-uuid",
  "is_staff": true,
  "idteam": "team-uuid",
  "mobile_access": true
}

Response: 201 Created
{
  "iduser": "new-user-uuid",
  "message": "User created successfully"
}
```

### 5.3 Get Leave Requests

```
GET /api/v2/get-leaves-working-days/{tenantId}
Authorization: Bearer {token}

Response: 200 OK
{
  "data": [
    {
      "iduser": "user-uuid",
      "user_name": "John Smith",
      "leaves": [
        {
          "id": "leave-uuid",
          "start_date": "2025-12-25",
          "end_date": "2025-12-26",
          "leave_type": "annual",
          "status": "approved"
        }
      ],
      "toil_balance": 8.5,
      "overtime_hours": 12
    }
  ]
}
```

### 5.4 Create/Update Leave

```
POST /api/v2/leaves
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "iduser": "user-uuid",
  "leave_type": "annual",
  "start_date": "2025-12-25",
  "end_date": "2025-12-26",
  "reason": "Christmas holiday"
}

Response: 201 Created
{
  "id": "leave-uuid",
  "status": "pending"
}
```

---

## 6. Permission Evaluation

### 6.1 Permission Check Logic

```typescript
// logic.ts
export function hasPermission(
  userPermissions: PermissionSet,
  entity: string,
  action: string
): boolean {
  const rolePerms = userPermissions[entity];
  if (!rolePerms) return false;
  return rolePerms.includes(action) || rolePerms.includes('full');
}
```

### 6.2 Usage in Components

```typescript
// In Vue component
import { usePermissionStore, PERMISSIONS } from '@/store/permission'

const permissionStore = usePermissionStore()
const canCreateUser = permissionStore.hasPermission(
  PERMISSIONS.TEAM.CREATE_USER
)
```

---

## 7. Sequence Diagrams

### 7.1 Leave Request Flow

```
┌──────┐    ┌────────────┐    ┌─────────┐    ┌─────────┐
│ Staff│    │  MyTeam    │    │  API    │    │ Manager │
└──┬───┘    └─────┬──────┘    └────┬────┘    └────┬────┘
   │  Submit Leave │               │              │
   │──────────────>│               │              │
   │               │ POST /leaves  │              │
   │               │──────────────>│              │
   │               │               │ Notify       │
   │               │               │─────────────>│
   │               │               │              │ Review
   │               │               │<─────────────│
   │               │               │ Approve/Reject│
   │               │<──────────────│              │
   │ Notification  │               │              │
   │<──────────────│               │              │
```

---

## 8. Key Methods

### 8.1 my-team.vue (39 methods)

| Method | Description |
|:---|:---|
| `getLeaves(users)` | Load leave data for users |
| `saveLeave(data)` | Create new leave request |
| `updateLeave(data, id)` | Update existing leave |
| `deleteLeave(data)` | Remove leave request |
| `openStaffProfile(el)` | Open staff detail modal |
| `createToil(data)` | Create TOIL entry |
| `deleteToil(id)` | Remove TOIL entry |
| `addPlannedWorkingDay(data)` | Add planned schedule |
| `fetchPublicHolidays()` | Load bank holidays |

### 8.2 user.vue (32 methods)

| Method | Description |
|:---|:---|
| `saveCreate(data)` | Create new user |
| `saveEdit(data)` | Update user |
| `confirmDelete()` | Delete user |
| `savePassword(data)` | Update password |
| `getMobileAccessCount()` | Check mobile limits |
| `assignTeamManagerToTeam(data)` | Set team manager |

---

## 9. Integration Points

| System | Type | Data Flow |
|:---|:---|:---|
| Schedule | Consumer | Staff availability |
| Reports | Consumer | Labor hours |
| Authentication | Provider | Login credentials |
| Subscription | Reference | Mobile access limits |

---

## 10. Performance Requirements

| Metric | Requirement |
|:---|:---|
| User list load | < 2 seconds |
| Leave calendar render | < 3 seconds |
| Permission check | < 10ms |
| Leave submission | < 1 second |

---

**Document End**
