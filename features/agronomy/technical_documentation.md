# Agronomy & Nutritional Management
## Technical Design Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-TSD-AGRO-001 |
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
│  Views              │  Components            │  Store (Pinia)   │
│  ├─ insights.vue    │  ├─ DiseaseInsights    │  ├─ agronomy.js  │
│  └─ nutritional.vue │  ├─ NutritionalInsight │  └─ spraying.js  │
│                     │  └─ InsightCard.vue    │                   │
├─────────────────────────────────────────────────────────────────┤
│                          API Layer                               │
│            apiClient + Dashboard API (Legacy)                    │
├─────────────────────────────────────────────────────────────────┤
│                     Backend (Node.js)                            │
│              Disease Models + Calculation Engine                 │
├─────────────────────────────────────────────────────────────────┤
│                      Database (MySQL)                            │
│      nutritional_input | spraying_session | product             │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Layer | Technology | Version |
|:---|:---|:---|
| Frontend Framework | Vue.js | 3.x |
| Chart Library | Chart.js | 4.x |
| State Management | Pinia | 2.x |
| Backend | Node.js / Express | 18.x |
| Disease Models | Custom algorithms | N/A |

---

## 2. Component Architecture

### 2.1 View Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `insights.vue` | `web/src/views/insights.vue` | Main insights dashboard with tabs |
| `nutritional.vue` | `web/src/views/nutritionalinput.vue` | Nutritional input management |

### 2.2 Insight Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `DiseaseInsights.vue` | `web/src/components/insights/DiseaseInsights/` | Disease risk visualization |
| `NutritionalInsight.vue` | `web/src/components/insights/NutritionalInsight/` | GP, GDD, Nitrogen charts |
| `InsightCard.vue` | `web/src/components/insights/InsightCard.vue` | KPI display widget |

### 2.3 Composables

| Composable | Path | Purpose |
|:---|:---|:---|
| `useFilterLists` | `web/src/components/insights/composables/` | Disease/nutrient filter options |
| `useDiseaseRisk` | `web/src/composables/useDiseaseRisk.js` | Risk calculation logic |
| `useNPKCalculator` | `web/src/composables/useNPKCalculator.js` | Nitrogen unit calculations |

---

## 3. State Management

### 3.1 Agronomy Store

**File**: `web/src/store/agronomy/agronomy.js`

```javascript
// State Structure
{
  diseaseData: [],        // Disease risk time series
  nutritionalData: [],    // Nutritional metrics
  selectedModel: null,    // Active disease model
  dateRange: {
    start: null,
    end: null
  },
  isLoading: false
}
```

### 3.2 Key Actions

| Action | Description | API Call |
|:---|:---|:---|
| `fetchDiseaseAnalysis(params)` | Load disease risk data | `GET /dashboardASingleData` |
| `fetchNutritionalAnalysis(params)` | Load GP/GDD data | `GET /dashboardASingleData` |
| `fetchSprayingRecords()` | Load application history | `GET /spraying-sessions` |

---

## 4. Data Models

### 4.1 Nutritional Input Entity

**Table**: `nutritional_input`

| Column | Type | Constraints | Description |
|:---|:---|:---|:---|
| `id` | CHAR(36) | PK | UUID identifier |
| `idtenant` | CHAR(36) | FK, NOT NULL | Tenant reference |
| `idproduct` | CHAR(36) | FK | Product reference |
| `quantity` | DECIMAL(10,4) | NOT NULL | Applied quantity |
| `quantity_unit` | VARCHAR(20) | | Unit (kg/ha, g/m²) |
| `application_date` | DATE | NOT NULL | Date of application |
| `idsite` | CHAR(36) | FK | Site reference |
| `nitrogen_units` | DECIMAL(8,4) | | Calculated N-units |
| `mode` | ENUM | | 'standard', 'advanced', 'precision' |

### 4.2 Spraying Session Entity

**Table**: `spraying_session`

| Column | Type | Constraints | Description |
|:---|:---|:---|:---|
| `session_token` | CHAR(64) | PK, UNIQUE | Session identifier |
| `idtenant` | CHAR(36) | FK | Tenant reference |
| `tractor_speed` | DECIMAL(6,2) | | Speed in km/h |
| `flow_rate` | DECIMAL(6,2) | | LPM value |
| `rpm` | INT | | Engine RPM |
| `nozzle_type` | VARCHAR(50) | | Nozzle identifier |
| `nozzle_spacing` | DECIMAL(6,2) | | Spacing in cm |
| `pressure` | DECIMAL(6,2) | | Bar value |
| `tank_size` | DECIMAL(8,2) | | Liters |

### 4.3 Disease Model Codes

| Code | Model Name | Algorithm |
|:---|:---|:---|
| `DOLLAR_SPOT` | Dollar Spot | Smith-Kerns |
| `FUSA_2` | Fusarium Patch | Custom |
| `ANTHRACNOSE` | Anthracnose | Stress Index |
| `RHIZOCTONIA` | Brown Patch | Temp/Humidity |
| `SMITH_KERN` | Smith-Kerns | Base algorithm |

---

## 5. API Specification

### 5.1 Disease Data Endpoint

```
GET /api/v2/dashboardASingleData(code={MODEL_CODE})
Authorization: Bearer {token}

Query Parameters:
- code: Disease model code (e.g., DAILY_DOLLAR_SPOT_PRED)
- startDate: ISO date
- endDate: ISO date

Response: 200 OK
{
  "DAILY_DOLLAR_SPOT_PRED": {
    "daily": {
      "value": 75.5,
      "unit": "%",
      "timestamp": "2025-12-30T08:00:00Z"
    },
    "history": [
      { "date": "2025-12-29", "value": 68.2 },
      { "date": "2025-12-28", "value": 55.0 }
    ]
  }
}
```

### 5.2 Nutritional Input Creation

```
POST /api/v2/nutritional-input
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "idproduct": "product-uuid",
  "quantity": 12.5,
  "quantity_unit": "kg/ha",
  "application_date": "2025-12-30",
  "site_ids": ["site-uuid"],
  "hole_ids": ["hole-uuid-1", "hole-uuid-2"],
  "mode": "advanced",
  "spraying_params": {
    "tractor_speed": 8.5,
    "tank_size": 600,
    "flow_rate": 2.5
  }
}

Response: 201 Created
{
  "id": "input-uuid",
  "nitrogen_units": 1.875,
  "tanks_required": 3,
  "total_volume": 1800
}
```

---

## 6. Calculation Logic

### 6.1 Tank Calculation

```
tanks_required = CEILING(total_water_volume / tank_size)
total_water_volume = dissolution_per_ha × total_area_ha
```

### 6.2 Nitrogen Units

```
n_units = (product_quantity × product_n_percentage) / 100
```

### 6.3 Recommended Flow Rate

```
flow_rate_lpm = (dissolution_per_ha × tractor_speed × spray_width) / 60000
```

### 6.4 Surface Area Pro-Rata

```
hole_quantity = total_quantity × (hole_surface / zone_surface)
hole_cost = total_cost × (hole_surface / zone_surface)
```

---

## 7. Sequence Diagrams

### 7.1 Disease Risk Fetch

```
┌──────┐      ┌─────────────────┐      ┌─────────┐      ┌─────────┐
│ User │      │ DiseaseInsights │      │  Store  │      │   API   │
└──┬───┘      └───────┬─────────┘      └────┬────┘      └────┬────┘
   │  Select Model    │                     │                │
   │─────────────────>│                     │                │
   │                  │  fetchDiseaseAnalysis()              │
   │                  │────────────────────>│                │
   │                  │                     │  GET /dashboard│
   │                  │                     │───────────────>│
   │                  │                     │   Risk Data    │
   │                  │                     │<───────────────│
   │                  │  Update Chart       │                │
   │                  │<────────────────────│                │
   │  Display Risk    │                     │                │
   │<─────────────────│                     │                │
```

---

## 8. Integration Points

| System | Type | Data Flow |
|:---|:---|:---|
| Weather Module | Inbound | Temp, humidity, leaf wetness for models |
| Stock Module | Outbound | Product consumption deduction |
| Map Module | Reference | Surface area for calculations |
| Reporting Module | Outbound | Treatment history data |

---

## 9. Performance Requirements

| Metric | Requirement |
|:---|:---|
| Disease chart load | < 3 seconds |
| Calculation response | < 500ms |
| Historical data range | Up to 365 days |

---

**Document End**
