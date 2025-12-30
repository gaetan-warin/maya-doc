# Alarms & Notifications
## Technical Design Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-TSD-ALRM-001 |
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
│  Views         │  Components              │  State              │
│  └─ alarms.vue │  ├─ AlarmForm.vue        │  (via piniaEndPoint)│
│                │  ├─ AlarmTable.vue       │                      │
│                │  ├─ AlarmsHistory.vue    │                      │
│                │  └─ AlarmSprayingNotif.  │                      │
├─────────────────────────────────────────────────────────────────┤
│                          API Layer                               │
│                    apiClient (Axios V2)                          │
├─────────────────────────────────────────────────────────────────┤
│                     Backend (Node.js)                            │
│     Alarm Service + Notification Service + History Service       │
├─────────────────────────────────────────────────────────────────┤
│                      Database (MySQL)                            │
│              alarm | alarm_history | notification_log            │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Layer | Technology | Version |
|:---|:---|:---|
| Frontend Framework | Vue.js | 3.x |
| State Management | Pinia (via piniaEndPoint) | 2.x |
| UI Components | CoreUI Vue | 4.x |
| Backend | Node.js / Express | 18.x |
| Notification | WhatsApp Business API, SMTP | - |

---

## 2. Component Architecture

### 2.1 View Component

| Component | Path | Responsibility |
|:---|:---|:---|
| `alarms.vue` | `web/src/views/alarms.vue` | Main controller, 347 lines |

### 2.2 Child Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `AlarmForm.vue` | `components/alarm/` | Alarm create/edit modal |
| `AlarmTable.vue` | `components/alarm/` | Active alarms list |
| `AlarmsHistory.vue` | `components/alarm/` | Triggered alarms log |
| `AlarmSprayingNotificationIndex.vue` | `components/alarm/sprayingNotification/` | Spraying reminder section |
| `AlarmSprayingNotificationForm.vue` | `components/alarm/sprayingNotification/` | Spraying notification form |
| `AlarmSprayingNotificationTable.vue` | `components/alarm/sprayingNotification/` | Spraying notifications list |

---

## 3. State Management

### 3.1 View State (alarms.vue)

```javascript
// Component data()
{
  api: piniaEndPoint(),
  setting: { open: false, save: false, edit: false },
  form: {
    rule_type: 'metric',          // 'metric' | 'incident'
    idalarm_name: null,           // Selected metric ID
    incident_name: null,          // Selected incident ID
    idalarm_type: null,           // Condition type ID
    value: ''                     // Threshold value
  },
  alarm: [],                      // Active alarms list
  alarmHistory: [],               // Triggered alarms log
  alarmType: [],                  // Condition options (<, =, >)
  alarmName: [],                  // Available metrics
  alarmMethod: [],                // Notification channels
  incidentsList: [],              // Available incident types
  loadingAlarm: false,
  loadingAlarmHistory: false
}
```

### 3.2 Key Methods

| Method | Description |
|:---|:---|
| `formatAlarm()` | Transform API response to table format |
| `formatAlarmHistory()` | Transform history API response |
| `formatAlarmName()` | Load available metrics |
| `initForm()` | Reset form to defaults |
| `openForm()` | Open create modal |
| `openEdit(e)` | Open edit modal with data |
| `save(e)` | Create or update alarm |
| `confirmDelete()` | Delete selected alarm |
| `getAlarmValue(min, max)` | Parse threshold value |

---

## 4. Data Models

### 4.1 Alarm Entity

**Table**: `alarm`

| Column | Type | Constraints | Description |
|:---|:---|:---|:---|
| `idalarm` | CHAR(36) | PK | UUID identifier |
| `idtenant` | CHAR(36) | FK, NOT NULL | Tenant reference |
| `idalarm_name` | CHAR(36) | FK | Metric reference (if metric type) |
| `incident_name` | CHAR(36) | FK | Incident type (if incident type) |
| `rule_type` | ENUM | | 'metric' or 'incident' |
| `idalarm_type` | CHAR(36) | FK | Condition type (<, =, >) |
| `value_min` | DECIMAL(10,2) | | Lower bound (if range) |
| `value_max` | DECIMAL(10,2) | | Upper bound / threshold |
| `is_active` | TINYINT | | Enabled flag |
| `created_at` | TIMESTAMP | | Creation timestamp |

### 4.2 Alarm Type Entity (Conditions)

**Table**: `alarm_type`

| Column | Type | Description |
|:---|:---|:---|
| `idalarm_type` | CHAR(36) | UUID identifier |
| `key` | VARCHAR(50) | Internal key |
| `label` | VARCHAR(100) | Display label |

**Condition Types:**
- `lower` → "< is less than"
- `equal` → "= is equal to"
- `greater` → "> is greater than"

### 4.3 Alarm Name Entity (Metrics)

**Table**: `alarm_name`

| Column | Type | Description |
|:---|:---|:---|
| `idalarm_name` | CHAR(36) | UUID identifier |
| `key` | VARCHAR(100) | Internal key |
| `label` | VARCHAR(100) | Display label |
| `is_forecast` | TINYINT | Forecast metric flag |

### 4.4 Alarm History Entity

**Table**: `alarm_history`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `idalarm` | CHAR(36) | Alarm reference |
| `idtenant` | CHAR(36) | Tenant reference |
| `triggered_at` | TIMESTAMP | Trigger timestamp |
| `triggered_value` | DECIMAL(10,2) | Actual value |
| `device_name` | VARCHAR(100) | Sensor/source |
| `notification_method` | ENUM | 'email', 'whatsapp', 'both' |
| `notification_sent` | TINYINT | Delivery confirmation |

---

## 5. API Specification

### 5.1 Get Active Alarms

```
GET /api/v2/getAlarm?idtenant={tenant}
Authorization: Bearer {token}

Response: 200 OK
[
  {
    "idalarm": "alarm-uuid",
    "alarm_name": "Daily Rainfall",
    "alarm_type": "> is greater than",
    "value_max": 25,
    "rule_type": "metric",
    "is_active": 1
  }
]
```

### 5.2 Create/Update Alarm

```
POST /api/v2/alarmCUD?idtenant={tenant}
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "rule_type": "metric",
  "idalarm_name": "metric-uuid",
  "idalarm_type": "type-uuid",
  "value": 25
}

Response: 201 Created
{
  "idalarm": "new-alarm-uuid",
  "message": "Alarm created successfully"
}
```

### 5.3 Delete Alarm

```
DELETE /api/v2/alarmCUD?idtenant={tenant}&idalarm={id}
Authorization: Bearer {token}

Response: 200 OK
{
  "message": "Alarm deleted successfully"
}
```

### 5.4 Get Alarm History

```
GET /api/v2/getAlarmHistory?idtenant={tenant}
Authorization: Bearer {token}

Response: 200 OK
[
  {
    "id": "history-uuid",
    "alarm_name": "Daily Rainfall",
    "triggered_at": "2025-12-30T10:30:00Z",
    "triggered_value": 28.5,
    "device_name": "Weather Station 1",
    "notification_method": "email"
  }
]
```

---

## 6. Sequence Diagrams

### 6.1 Metric Alarm Trigger Flow

```
┌──────────┐    ┌──────────┐    ┌─────────┐    ┌──────────┐    ┌──────────┐
│  Sensor  │    │  Backend │    │ Alarm   │    │  Notif   │    │   User   │
│  Data    │    │  Ingress │    │ Service │    │  Service │    │          │
└────┬─────┘    └────┬─────┘    └────┬────┘    └────┬─────┘    └────┬─────┘
     │  Send Data    │               │              │               │
     │──────────────>│               │              │               │
     │               │ Check Alarms  │              │               │
     │               │──────────────>│              │               │
     │               │               │ Match Found  │               │
     │               │               │ Log History  │               │
     │               │               │─────────────>│               │
     │               │               │              │ Send Email    │
     │               │               │              │──────────────>│
     │               │               │              │ Send WhatsApp │
     │               │               │              │──────────────>│
```

---

## 7. Notification Implementation

### 7.1 Email Notification

- Uses SMTP service configured at server level
- Template includes: Alarm name, value, timestamp, site name
- Sent to user's registered email address

### 7.2 WhatsApp Notification

- Integrates with WhatsApp Business API via Maya Bot
- Requires user phone number in account settings
- Includes quick action links where applicable

---

## 8. Integration Points

| System | Type | Data Flow |
|:---|:---|:---|
| Weather Service | Inbound | Metric values for evaluation |
| Agronomy Service | Inbound | Disease risk values |
| Reports Module | Inbound | Incident creation events |
| User Settings | Reference | Notification preferences |
| Spraying Calendar | Reference | Task scheduling |

---

## 9. Performance Requirements

| Metric | Requirement |
|:---|:---|
| Alarm list load | < 2 seconds |
| Alarm creation | < 1 second |
| History load | < 3 seconds |
| Notification delivery | < 30 seconds from trigger |

---

**Document End**
