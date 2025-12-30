# Reports & Analytics
## Technical Design Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-TSD-ANLY-001 |
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
│  Views                │  Components              │  Stores       │
│  ├─ soil_analysis.vue │  ├─ SoilModeIndex.vue   │  graphBuilder │
│  ├─ graphbuilder.vue  │  ├─ GraphBuilder.vue    │               │
│  └─ history.vue       │  ├─ GraphStats.vue      │               │
│                       │  ├─ GraphTabs.vue       │               │
│                       │  └─ HistoryTable.vue    │               │
├─────────────────────────────────────────────────────────────────┤
│                          API Layer                               │
│                    apiClient (Axios V2)                          │
├─────────────────────────────────────────────────────────────────┤
│                     Backend (Node.js)                            │
│        Analytics Service + History Service + Export Service      │
├─────────────────────────────────────────────────────────────────┤
│                      Database (MySQL)                            │
│    soil_analysis | graph_template | history | summary_tables    │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Layer | Technology | Version |
|:---|:---|:---|
| Frontend Framework | Vue.js | 3.x (Composition API) |
| State Management | Pinia | 2.x |
| Charting | Chart.js / ApexCharts | Latest |
| Date Picker | VueDatePicker | 3.x |
| Backend | Node.js / Express | 18.x |

---

## 2. Component Architecture

### 2.1 View Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `soil_analysis.vue` | `web/src/views/` | Container for SoilMode component |
| `graphbuilder.vue` | `web/src/views/` | Graph/Stats tab controller |
| `history.vue` | `web/src/views/` | Activity history (1011 lines) |

### 2.2 Graph Builder Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `GraphBuilder.vue` | `components/graphBuilder/` | Main chart visualization |
| `GraphStats.vue` | `components/graphBuilder/` | Statistical summary view |
| `GraphTabs.vue` | `components/graphBuilder/GraphTabs/` | Tab navigation |
| `GraphSummary.vue` | `components/graphBuilder/GraphSummary/` | Summary cards |
| `MixGraph.vue` | `components/graphBuilder/` | Multi-metric chart |

### 2.3 Soil Analysis Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `SoilModeIndex.vue` | `components/analysis/SoilMode/` | Main soil analysis container |
| `SoilModeGraphPh.vue` | `components/analysis/SoilMode/PHREDOX/` | pH visualization |

### 2.4 Composables

| Composable | Path | Purpose |
|:---|:---|:---|
| `useGraphBuilderHandler.js` | `components/graphBuilder/composables/` | Chart event handling |
| `useGraphData.js` | `components/graphBuilder/composables/` | Data fetching logic |

---

## 3. State Management

### 3.1 Graph Builder Store

**File**: `web/src/store/graphBuilder/graphBuilder.js`

```javascript
// State Structure
{
  activeTab: 'graph',              // 'graph' | 'data'
  viewMode: false,                 // Template mode
  setting: {
    open: false,
    save: false,
    editId: null,
    deleteId: null,
    openModalDelete: false,
    confirmDelete: false
  },
  sites: [],                       // Available sites
  loading: false,
  
  // Metric data
  environmentMetrics: null,
  performanceMetrics: null,
  nutritionalInputs: null,
  tasks: null
}
```

### 3.2 Key Actions

| Action | Description | API Call |
|:---|:---|:---|
| `setTab(tab)` | Switch active tab | - |
| `setViewMode(val)` | Toggle template mode | - |
| `updateSetting(payload)` | Update settings | - |
| `fetchEnvironmentMetrics(payload)` | Load env data | `POST /graph-builder-data` |
| `fetchPerformanceMetrics(payload)` | Load perf data | `POST /graph-builder-data` |
| `fetchNutritionalInputs(payload)` | Load nutrition data | `POST /graph-builder-data` |
| `fetchTasks(payload)` | Load task data | `POST /graph-builder-data` |

---

## 4. Data Models

### 4.1 Soil Analysis Entity

**Table**: `soil_analysis`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `idtenant` | CHAR(36) | Tenant reference |
| `idsite` | CHAR(36) | Site reference |
| `analysis_date` | DATE | Analysis date |
| `name` | VARCHAR(255) | Analysis name |
| `type` | VARCHAR(100) | Analysis type |
| `substance` | VARCHAR(100) | Tested substance |
| `depth` | VARCHAR(50) | Sample depth |
| `value` | DECIMAL(10,4) | Result value |
| `unit` | VARCHAR(20) | Measurement unit |
| `pdf_url` | VARCHAR(500) | PDF result path |

### 4.2 Graph Template Entity

**Table**: `graph_template`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `idtenant` | CHAR(36) | Tenant reference |
| `iduser` | CHAR(36) | User who created |
| `name` | VARCHAR(100) | Template name |
| `config` | JSON | Chart configuration |
| `created_at` | TIMESTAMP | Creation timestamp |

### 4.3 History Query Model

History uses aggregated data from `summary_tasks` table:

| Column | Type | Description |
|:---|:---|:---|
| `date` | DATE | Activity date |
| `type` | VARCHAR(100) | Activity type |
| `site` | VARCHAR(255) | Site name |
| `zones` | TEXT | Affected zones |
| `staff` | TEXT | Assigned staff |
| `activity` | VARCHAR(255) | Activity description |
| `cost` | DECIMAL(12,2) | Total cost |
| `duration` | DECIMAL(10,2) | Duration in hours |

---

## 5. API Specification

### 5.1 Graph Builder Data

```
POST /api/v2/graph-builder-data
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "idtenant": "tenant-uuid",
  "type": "environment",
  "startDate": "2025-01-01",
  "endDate": "2025-12-30",
  "sites": ["site-uuid"],
  "metrics": ["air_temperature", "humidity"],
  "frequency": "daily"
}

Response: 200 OK
{
  "data": [
    {
      "date": "2025-12-30",
      "air_temperature": 15.5,
      "humidity": 65.2
    }
  ]
}
```

### 5.2 Get History

```
GET /api/v2/history?idtenant={tenant}&start={date}&end={date}
Authorization: Bearer {token}

Query Parameters:
- idtenant: UUID (required)
- start: date (required)
- end: date (required)
- sites: comma-separated UUIDs (optional)
- type: activity type (optional)
- areas: comma-separated UUIDs (optional)
- activity: activity filter (optional)

Response: 200 OK
{
  "records": [
    {
      "date": "2025-12-30",
      "type": "Mowing",
      "site": "Main Course",
      "zones": "Hole 1, Hole 2",
      "staff": "John Smith",
      "activity": "Regular mowing",
      "cost": 150.00,
      "duration": 4.5
    }
  ],
  "kpi": {
    "totalCost": 15000.00,
    "totalHours": 450,
    "taskCount": 120
  }
}
```

### 5.3 Get Soil Analysis

```
GET /api/v2/soil-analysis?idtenant={tenant}
Authorization: Bearer {token}

Query Parameters:
- substance: filter by substance
- site: filter by site
- area: filter by area
- depth: filter by depth

Response: 200 OK
{
  "data": [
    {
      "id": "analysis-uuid",
      "analysis_date": "2025-06-15",
      "site": "Main Course",
      "name": "Spring Analysis",
      "type": "Full Spectrum",
      "pdf_url": "/uploads/soil/analysis-123.pdf"
    }
  ]
}
```

---

## 6. Sequence Diagrams

### 6.1 Graph Builder Data Flow

```
┌──────┐    ┌────────────┐    ┌─────────┐    ┌─────────┐
│ User │    │ GraphBuilder│   │  Store  │    │   API   │
└──┬───┘    └─────┬──────┘    └────┬────┘    └────┬────┘
   │  Select Metrics  │              │              │
   │─────────────────>│              │              │
   │                  │ fetchEnvironmentMetrics()  │
   │                  │─────────────>│              │
   │                  │              │ POST /graph-builder-data
   │                  │              │─────────────>│
   │                  │              │  Data[]      │
   │                  │              │<─────────────│
   │                  │  Render Chart│              │
   │                  │<─────────────│              │
   │  View Chart      │              │              │
   │<─────────────────│              │              │
```

---

## 7. History Component Methods

### 7.1 Key Computed Properties (46 total)

| Method | Description |
|:---|:---|
| `selectedTypeId()` | Get selected filter type |
| `kpi()` | Calculate KPI summary |
| `filterItems()` | Apply all filters to data |
| `filterSites()` | Get available sites |
| `filtertype()` | Get available types |
| `filterAreas()` | Get available areas |
| `filterActivity()` | Get available activities |
| `filterTableStaff()` | Filter staff view data |
| `tableOne()` | Build report view table |
| `staffs()` | Build staff view table |

### 7.2 Key Methods

| Method | Description |
|:---|:---|
| `request()` | Fetch history data |
| `exportToCSV()` | Generate CSV export |
| `print()` | Trigger print dialog |

---

## 8. Export Implementation

### 8.1 CSV Export

```javascript
exportToCSV() {
  const data = this.filterItems.map(item => ({
    Date: item.date,
    Type: item.type,
    Site: item.site,
    Zones: item.zones,
    Staff: item.staff,
    Activity: item.activity,
    Cost: item.cost,
    Duration: item.duration
  }));
  
  const csv = this.convertToCSV(data);
  this.downloadFile(csv, 'history.csv', 'text/csv');
}
```

---

## 9. Integration Points

| System | Type | Data Flow |
|:---|:---|:---|
| Weather Service | Inbound | Environmental metrics |
| Agronomy Service | Inbound | Performance metrics |
| Task Service | Inbound | Activity and labor data |
| Fleet Service | Inbound | Machine usage metrics |
| Water Service | Inbound | Irrigation flow data |
| Stock Service | Inbound | Inventory movements |

---

## 10. Performance Requirements

| Metric | Requirement |
|:---|:---|
| Graph Builder load | < 5 seconds (30-day range) |
| History load | < 3 seconds (1000 records) |
| Interactive Report load | < 5 seconds |
| Export generation | < 10 seconds |

---

**Document End**
