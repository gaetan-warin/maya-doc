# Reports & Incident Management
## Technical Design Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-TSD-REPT-001 |
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
│  Views         │  Components          │  Store (Pinia)          │
│  └─ reports.vue│  ├─ ReportForm.vue   │  └─ reports.js          │
│                │  ├─ ReportTable.vue  │                          │
│                │  ├─ ReportMap.vue    │                          │
│                │  └─ IncidentsIndex   │                          │
├─────────────────────────────────────────────────────────────────┤
│                          API Layer                               │
│                    apiClient (Axios V2)                          │
├─────────────────────────────────────────────────────────────────┤
│                     Backend (Node.js)                            │
│              Reports Service + Follow-up Service                 │
├─────────────────────────────────────────────────────────────────┤
│                      Database (MySQL)                            │
│           reports | report_followup | taggable | tag             │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Layer | Technology | Version |
|:---|:---|:---|
| Frontend Framework | Vue.js | 3.x |
| State Management | Pinia | 2.x |
| Map Provider | Google Maps API | Latest |
| Export Library | xlsx | 0.18.x |
| Backend | Node.js / Express | 18.x |

---

## 2. Component Architecture

### 2.1 View Component

| Component | Path | Responsibility |
|:---|:---|:---|
| `reports.vue` | `web/src/views/reports.vue` | Main controller, 805 lines, 42 methods |

### 2.2 Child Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `ReportForm.vue` | `web/src/components/reports/` | Report create/edit modal |
| `ReportTable.vue` | `web/src/components/reports/` | Data table display |
| `ReportMap.vue` | `web/src/components/reports/` | Map marker placement |
| `IncidentsIndex.vue` | `web/src/components/reports/incidents/` | Incident type management |

---

## 3. State Management

### 3.1 Reports Store

**File**: `web/src/store/reports.js`

```javascript
// State Structure
{
  api: piniaEndPoint(),     // API endpoint reference
  reports: [],              // Report list
  incidents: [],            // Incident type list
  activeIncidents: [],      // Active incident types
  followups: [],            // Follow-up actions
  isLoading: false,
  total: 0                  // Pagination total
}
```

### 3.2 Key Actions (20 total)

| Action | Description | API Call |
|:---|:---|:---|
| `getReports()` | Load all reports | `GET /reports` |
| `getIncident()` | Load incident types | `GET /incidents` |
| `saveReportsCUD(data, groupId)` | Create/update report | `POST /reportCUD` |
| `deleteReport(idtenant, payload)` | Delete report | `DELETE /report` |
| `getAllReports(tenant, page, search, ...)` | Paginated reports | `GET /reports/all` |
| `getAllReportsExport(tenant)` | Export data | `GET /reports/export` |
| `saveFollowupForm(payload, tenant)` | Create follow-up | `POST /followup` |
| `updateFollowupForm(payload, tenant)` | Update follow-up | `PUT /followup` |
| `changeFollowupStatus(payload)` | Change status | `PATCH /followup/status` |
| `getFollowupForReport(id)` | Get report follow-ups | `GET /followup/{id}` |
| `getReportSetting(tenant)` | Get settings | `GET /report-setting` |

---

## 4. Data Models

### 4.1 Report Entity

**Table**: `reports`

| Column | Type | Constraints | Description |
|:---|:---|:---|:---|
| `id` | CHAR(36) | PK | UUID identifier |
| `idtenant` | CHAR(36) | FK, NOT NULL | Tenant reference |
| `idgroup` | CHAR(36) | FK | Group reference |
| `reportedon` | DATE | NOT NULL | Report date |
| `idtype` | CHAR(36) | FK | Incident type reference |
| `idsite` | CHAR(36) | FK | Site reference |
| `idsubsite` | CHAR(36) | FK | Sub-site reference |
| `latitude` | DECIMAL(10,8) | | GPS latitude |
| `longitude` | DECIMAL(11,8) | | GPS longitude |
| `severity` | TINYINT | | 1=Low, 2=Medium, 3=High |
| `status` | ENUM | | 'reported', 'in_progress', 'resolved', 'closed' |
| `comments` | TEXT | | Free-text description |
| `image` | VARCHAR(255) | | Image path/URL |
| `created_at` | TIMESTAMP | | Creation timestamp |
| `updated_at` | TIMESTAMP | | Last update |

### 4.2 Report Follow-up Entity

**Table**: `report_followup`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `report_id` | CHAR(36) | Parent report reference |
| `action_date` | DATE | Follow-up date |
| `description` | TEXT | Action description |
| `status` | ENUM | Status after action |
| `user_id` | CHAR(36) | User who performed |
| `created_at` | TIMESTAMP | Creation timestamp |

### 4.3 Tag System (Incident Types)

**Table**: `tag`

| Column | Type | Description |
|:---|:---|:---|
| `idtag` | CHAR(36) | UUID identifier |
| `label` | VARCHAR(100) | Display name |
| `idtag_category` | CHAR(36) | Category reference |
| `is_active` | TINYINT | Active flag |

**Table**: `taggable` (Polymorphic relationship)

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `idtag` | CHAR(36) | Tag reference |
| `taggable_id` | CHAR(36) | Referenced entity ID |
| `taggable_type` | VARCHAR(50) | Entity type ('report', 'zone', etc.) |

---

## 5. API Specification

### 5.1 Get All Reports (Paginated)

```
GET /api/v2/reports/all
Authorization: Bearer {token}

Query Parameters:
- idtenant: UUID (required)
- page: number
- per_page: number (default 10)
- search: string
- sortkey: column name
- sortorder: 'asc' | 'desc'

Response: 200 OK
{
  "data": [
    {
      "id": "report-uuid",
      "reportedon": "2025-12-30",
      "type": { "idtag": "type-uuid", "label": "Dollar Spot" },
      "site": { "name": "Main Course" },
      "subsite": { "name": "Hole 1" },
      "severity": 2,
      "status": "reported",
      "image": "/uploads/reports/img123.jpg"
    }
  ],
  "total": 150
}
```

### 5.2 Create/Update Report

```
POST /api/v2/reportCUD
Content-Type: multipart/form-data
Authorization: Bearer {token}

Form Data:
- reportedon: date
- idtype: UUID
- idsite: UUID
- idsubsite: UUID (optional)
- zones: JSON array of zone IDs
- severity: 1|2|3
- status: string
- comments: string
- image: file (optional)
- latitude: decimal (optional)
- longitude: decimal (optional)

Response: 201 Created
{
  "id": "report-uuid",
  "message": "Report saved successfully"
}
```

### 5.3 Create Follow-up

```
POST /api/v2/followup
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "report_id": "report-uuid",
  "action_date": "2025-12-30",
  "description": "Applied fungicide treatment",
  "status": "in_progress"
}

Response: 201 Created
{
  "id": "followup-uuid"
}
```

### 5.4 Export Reports

```
GET /api/v2/reports/export?idtenant={tenant}
Authorization: Bearer {token}

Response: 200 OK
{
  "data": [
    // All reports without pagination
  ]
}
```

---

## 6. Sequence Diagrams

### 6.1 Report Creation Flow

```
┌──────┐      ┌────────────┐      ┌─────────┐      ┌─────────┐      ┌────────┐
│ User │      │ ReportForm │      │  Store  │      │   API   │      │ Upload │
└──┬───┘      └─────┬──────┘      └────┬────┘      └────┬────┘      └───┬────┘
   │  Fill Form     │                  │                │               │
   │───────────────>│                  │                │               │
   │  Select Marker │                  │                │               │
   │───────────────>│                  │                │               │
   │  Upload Image  │                  │                │               │
   │───────────────>│                  │                │               │
   │                │  Process Image   │                │               │
   │                │─────────────────────────────────────────────────>│
   │                │                  │                │   Upload URL  │
   │                │<─────────────────────────────────────────────────│
   │  Submit        │                  │                │               │
   │───────────────>│                  │                │               │
   │                │  saveReportsCUD()│                │               │
   │                │─────────────────>│                │               │
   │                │                  │  POST /reportCUD               │
   │                │                  │───────────────>│               │
   │                │                  │  201 Created   │               │
   │                │                  │<───────────────│               │
   │                │  Refresh List    │                │               │
   │                │<─────────────────│                │               │
```

---

## 7. Export Implementation

### 7.1 Excel Export

```javascript
import * as XLSX from 'xlsx';

exportToExcel() {
  const exportData = this.reports.map(r => ({
    Date: r.reportedon,
    Type: r.type?.label,
    Course: this.courseName(r),
    Area: this.getAreaExport(r),
    Severity: this.getSeverityLabel(r.severity),
    Status: r.status,
    Comments: r.comments
  }));
  
  const ws = XLSX.utils.json_to_sheet(exportData);
  const wb = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(wb, ws, 'Reports');
  XLSX.writeFile(wb, `Reports_${new Date().toISOString()}.xlsx`);
}
```

---

## 8. Integration Points

| System | Type | Data Flow |
|:---|:---|:---|
| Maps Module | Bidirectional | Report markers on map |
| AI/Disease Models | Outbound | Disease reports for training |
| WhatsApp Bot | Inbound | Field-submitted reports |
| Schedule Module | Bidirectional | Follow-ups displayed in Schedule view |

---

## 9. Schedule Integration Architecture

### 9.1 Overview

Follow-up actions are displayed in the Schedule view but do **NOT** create entries in the `task` table. Instead, they use a separate notification system.

### 9.2 Data Flow

```
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────────────────┐
│  Report Form    │────▶│    ReportService     │────▶│  TaskPushNotification   │
│  (Follow-up)    │     │                      │     │       Event             │
└─────────────────┘     └──────────────────────┘     └───────────┬─────────────┘
                                                                 │
                                                                 ▼
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────────────────┐
│  Schedule View  │◀────│  ReportController    │◀────│  task_push_notification │
│                 │     │  (getFollowupRecords)│     │        table            │
└─────────────────┘     └──────────────────────┘     └─────────────────────────┘
```

### 9.3 Task Push Notification Table

**Table**: `task_push_notification`

| Column | Type | Description |
|:---|:---|:---|
| `id` | INT | Primary key |
| `idgroup` | BINARY(16) | Group reference |
| `idtenant` | BINARY(16) | Tenant reference |
| `idsites` | JSON | Site IDs array |
| `idstaffs` | JSON | Assigned staff IDs |
| `idholes` | JSON | Hole/subsite IDs |
| `idareas` | JSON | Area IDs |
| `idrecord` | BINARY(16) | Original follow-up ID |
| `task_type` | VARCHAR | `'incident'`, `'schedule'`, `'spraying'` |
| `task_title` | VARCHAR | Display title (incident type label) |
| `task_start_date` | DATE | Scheduled date |
| `task_start_time` | TIME | Scheduled time |
| `merge_key` | VARCHAR | Grouping key for deduplication |
| `status` | TINYINT | 0=pending, 1=sent |
| `notification_status` | TINYINT | Push notification delivery status |
| `created_at` | TIMESTAMP | Creation timestamp |

### 9.4 Dual Table Architecture

The system uses **two parallel task systems**:

| Table | Purpose | Created By |
|:---|:---|:---|
| `task` | Primary schedule tasks | Schedule module, templates |
| `task_push_notification` | Follow-ups, incidents, spraying | Reports, Spraying modules |

The Schedule view fetches from both tables and merges results for unified display.

### 9.5 Schedule Integration Endpoints

| Endpoint | Method | Description |
|:---|:---|:---|
| `/get-Followup-for-schedule/{idGroup}/{date}` | GET | Retrieve follow-ups for Schedule display |
| `/update-Followup-schedule-status/{followupId}` | PUT | Update follow-up status from Schedule |
| `/save-dwd-setting-for-followup` | POST | Configure Digital White Board follow-up settings |

### 9.6 Key Backend Files

| File | Responsibility |
|:---|:---|
| `app/Services/ReportService.php` | Dispatches TaskPushNotification events |
| `app/Services/TaskPushNotificationService.php` | Saves to task_push_notification table |
| `app/Helpers/PushNotificationHelper.php` | Formats follow-up data for notifications |
| `app/Events/TaskPushNotification.php` | Event class for notification dispatch |
| `app/Models/TaskPushNotification.php` | Eloquent model for the table |

### 9.7 Status Synchronization

Follow-up status is tracked in **two places**:

| Table | Column | Values |
|:---|:---|:---|
| `report` | `follow_up_status` | 1=open, 2=in_progress, 3=completed |
| `report_follow_up` | `status` | 'todo', 'completed' |

Both are updated when status changes via either Reports page or Schedule page.

### 9.8 Push Notification Deduplication

The system prevents duplicate notifications using cache-based deduplication:

```php
$notificationKey = 'task_notification_' . $reportId . '_' . $followUpStatus . '_' . $userId;
Cache::put($cacheKey, true, 30); // 30 second window
```

---

## 10. Performance Requirements

| Metric | Requirement |
|:---|:---|
| Report list load | < 2 seconds (100 records) |
| Report creation | < 3 seconds (with image) |
| Export generation | < 5 seconds (1000 records) |
| Map marker render | < 1 second |

---

**Document End**
