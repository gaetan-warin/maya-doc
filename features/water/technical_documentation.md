# Water & Irrigation Management
## Technical Design Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-TSD-WATER-001 |
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
│  Views         │  Components            │  Store (Pinia)        │
│  └─ water.vue  │  ├─ WaterIndex.vue     │  └─ water.js          │
│                │  ├─ WaterInformatics   │                        │
│                │  ├─ WaterForm.vue      │                        │
│                │  └─ InformaticsCard    │                        │
├─────────────────────────────────────────────────────────────────┤
│                          API Layer                               │
│              apiClient + Dashboard API (for ET/Rainfall)         │
├─────────────────────────────────────────────────────────────────┤
│                     Backend (Node.js)                            │
│                Water Sources + Readings + Statistics             │
├─────────────────────────────────────────────────────────────────┤
│                      Database (MySQL)                            │
│         water_source | water_reading | water_source_type        │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Layer | Technology | Version |
|:---|:---|:---|
| Frontend Framework | Vue.js | 3.x |
| State Management | Pinia | 2.x |
| Backend | Node.js / Express | 18.x |
| Database | MySQL | 8.x |

---

## 2. Component Architecture

### 2.1 View Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `water.vue` | `web/src/views/water.vue` | Wrapper view |
| `WaterIndex.vue` | `web/src/components/water/WaterIndex.vue` | Main controller, readings table |

### 2.2 Child Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `WaterInformatics.vue` | `web/src/components/water/WaterInformatics.vue` | Dashboard KPIs |
| `WaterForm.vue` | `web/src/components/water/WaterForm.vue` | Reading entry modal |
| `InformaticsCard.vue` | `web/src/components/water/InformaticsCard.vue` | Individual KPI widget |
| `WaterChart.vue` | `web/src/components/water/WaterChart.vue` | Consumption visualization |

### 2.3 Composables

| Composable | Path | Purpose |
|:---|:---|:---|
| `useWaterUnits` | `web/src/composables/useWaterUnits.js` | Unit conversion (m³/gallons) |
| `useWaterFormState` | `web/src/composables/useWaterFormState.js` | Form state management |

---

## 3. State Management

### 3.1 Water Store

**File**: `web/src/store/water.js`

```javascript
// State Structure
{
  waterSources: [],        // Source configuration
  readings: [],            // Water readings
  statistics: {},          // Aggregated stats
  rainfallLast24h: 0,      // mm
  etLast24h: 0,            // mm
  waterBalance: 0,         // Calculated balance
  isLoading: false,
  isLoadingInformatics: false
}
```

### 3.2 Key Actions

| Action | Description | API Call |
|:---|:---|:---|
| `fetchWaterSources()` | Load source config | `GET /settings/water-sources` |
| `fetchWaterReadings()` | Load readings | `GET /water/readings` |
| `fetchWaterStatistics(siteId)` | Load aggregated stats | `GET /water/statistics` |
| `fetchETAndRainfall()` | Load weather data | `GET /dashboardASingleData` |
| `createReading(payload)` | Add new reading | `POST /water/readings` |
| `updateReading(id, payload)` | Update reading | `PUT /water/readings/{id}` |
| `deleteReading(id)` | Remove reading | `DELETE /water/readings/{id}` |

---

## 4. Data Models

### 4.1 Water Source Entity

**Table**: `water_source`

| Column | Type | Constraints | Description |
|:---|:---|:---|:---|
| `id` | CHAR(36) | PK | UUID identifier |
| `idtenant` | CHAR(36) | FK, NOT NULL | Tenant reference |
| `name` | VARCHAR(100) | NOT NULL | Source name |
| `source_type_id` | CHAR(36) | FK | Type reference |
| `is_legacy` | TINYINT | DEFAULT 0 | Legacy flag |
| `created_at` | TIMESTAMP | | Creation timestamp |

### 4.2 Water Reading Entity

**Table**: `water_reading`

| Column | Type | Constraints | Description |
|:---|:---|:---|:---|
| `id` | CHAR(36) | PK | UUID identifier |
| `water_source_id` | CHAR(36) | FK, NOT NULL | Source reference |
| `idtenant` | CHAR(36) | FK, NOT NULL | Tenant reference |
| `reading_date` | DATE | NOT NULL | Reading date |
| `measurement_type` | ENUM | | 'meter_reading', 'consumption_value' |
| `reading_value` | DECIMAL(12,2) | NOT NULL | Raw value |
| `calculated_consumption` | DECIMAL(12,2) | | Derived consumption |
| `notes` | TEXT | | Optional notes |
| `created_at` | TIMESTAMP | | Creation timestamp |

### 4.3 Water Source Type Entity

**Table**: `water_source_type`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `name` | VARCHAR(50) | Type name |
| `code` | VARCHAR(20) | Type code |

**Standard Types**:
| Code | Name |
|:---|:---|
| `METER` | Meter Reading |
| `PUMP` | Pump System |
| `MANUAL` | Manual Entry |

---

## 5. API Specification

### 5.1 Get Water Statistics

```
GET /api/v2/water/statistics
Authorization: Bearer {token}

Query Parameters:
- idtenant: UUID (required)
- site_id: UUID (optional)

Response: 200 OK
{
  "monthly_consumption": 1250.5,
  "yearly_consumption": 15420.0,
  "days_watered": 18,
  "average_daily": 65.8
}
```

### 5.2 Create Water Reading

```
POST /api/v2/water/readings
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "water_source_id": "source-uuid",
  "reading_date": "2025-12-30",
  "measurement_type": "meter_reading",
  "reading_value": 45678.50,
  "notes": "End of month reading"
}

Response: 201 Created
{
  "id": "reading-uuid",
  "calculated_consumption": 125.5,
  "message": "Reading created successfully"
}
```

### 5.3 Fetch ET and Rainfall

```
GET /api/v2/dashboardASingleData(code=ETP24)
GET /api/v2/dashboardASingleData(code=LAST_24H_RAINFALL)
Authorization: Bearer {token}

Response: 200 OK
{
  "ETP24": {
    "daily": {
      "value": 5.8,
      "unit": "mm"
    }
  }
}
```

---

## 6. Calculation Logic

### 6.1 Water Balance

```javascript
async fetchETAndRainfall() {
  const [rainfallRes, etRes] = await Promise.all([
    apiClient.get('/dashboardASingleData(code=LAST_24H_RAINFALL)'),
    apiClient.get('/dashboardASingleData(code=ETP24)')
  ]);
  
  this.rainfallLast24h = rainfallRes?.data?.LAST_24H_RAINFALL?.daily?.value ?? 0;
  this.etLast24h = etRes?.data?.ETP24?.daily?.value ?? 0;
  this.waterBalance = this.rainfallLast24h - this.etLast24h;
}
```

### 6.2 Consumption Calculation (Meter Type)

```javascript
calculated_consumption = current_reading - previous_reading;

// Validation
if (calculated_consumption < 0) {
  throw new Error('Meter reading cannot be lower than previous');
}
```

### 6.3 Monthly Aggregation

```sql
SELECT 
  SUM(calculated_consumption) as monthly_consumption,
  COUNT(DISTINCT reading_date) as days_watered
FROM water_reading
WHERE idtenant = ?
  AND MONTH(reading_date) = MONTH(CURRENT_DATE)
  AND YEAR(reading_date) = YEAR(CURRENT_DATE);
```

---

## 7. Sequence Diagrams

### 7.1 Water Informatics Load

```
┌──────┐      ┌──────────────────┐      ┌─────────┐      ┌─────────┐
│ User │      │ WaterInformatics │      │  Store  │      │   API   │
└──┬───┘      └────────┬─────────┘      └────┬────┘      └────┬────┘
   │  View Page        │                     │                │
   │──────────────────>│                     │                │
   │                   │  fetchWaterStatistics()              │
   │                   │────────────────────>│                │
   │                   │                     │  GET /stats    │
   │                   │                     │───────────────>│
   │                   │                     │                │
   │                   │  fetchETAndRainfall()               │
   │                   │────────────────────>│                │
   │                   │                     │  GET /dashboard│
   │                   │                     │───────────────>│
   │                   │                     │  Calc Balance  │
   │                   │  Render Cards       │                │
   │                   │<────────────────────│                │
   │  Display KPIs     │                     │                │
   │<──────────────────│                     │                │
```

---

## 8. Security & Permissions

### 8.1 Permission Constants

```javascript
PERMISSIONS.WATER = {
  CREATE_NEW_RECORDS: 'water.create',
  EDIT_DELETE_RECORDS: 'water.edit'
}
```

### 8.2 Access Control

| Permission | UI Impact |
|:---|:---|
| `water.create` | Show "Add Reading" button |
| `water.edit` | Show row action buttons |

---

## 9. Performance Requirements

| Metric | Requirement |
|:---|:---|
| Statistics load | < 2 seconds |
| Reading creation | < 1 second |
| Balance calculation | < 500ms |

---

## 10. Dependencies

| Dependency | Type | Purpose |
|:---|:---|:---|
| Weather Module | Critical | ET and Rainfall for balance |
| Task Module | Integration | Irrigation task labor |
| Reporting Module | Consumer | Consumption reports |

---

**Document End**
