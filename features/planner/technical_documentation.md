# Planner Module
## Technical Design Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-TSD-PLNR-001 |
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
│  Views                     │  Components                │ Stores │
│  ├─ fertilisation_planner  │  ├─ FertilisationPlanner   │ fertil.│
│  ├─ budget_planner         │  ├─ ProductPlannerTable    │ budget │
│  ├─ daily-routine          │  ├─ Lists.vue (daily)      │ daily  │
│  ├─ spraying_routine       │  ├─ Lists.vue (spraying)   │ spray  │
│  └─ action-planner         │  ├─ MyTask.vue             │ action │
│                            │  └─ AnnualPlan.vue         │        │
├─────────────────────────────────────────────────────────────────┤
│                          API Layer                               │
│                    apiClient (Axios V2)                          │
├─────────────────────────────────────────────────────────────────┤
│                     Backend (Node.js)                            │
│  FertilisationService + BudgetService + RoutineService + Action │
├─────────────────────────────────────────────────────────────────┤
│                      Database (MySQL)                            │
│   fertilization_plan | budget_plan | routine | action_plan      │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Layer | Technology | Version |
|:---|:---|:---|
| Frontend Framework | Vue.js | 3.x (Composition API) |
| State Management | Pinia | 2.x |
| Date Handling | Moment.js | 2.x |
| UI Components | CoreUI Vue + Custom | 4.x |

---

## 2. Component Architecture

### 2.1 View Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `fertilisation_planner.vue` | `views/` | Tabs: Product Composition + Planning |
| `budget_planner.vue` | `views/` | Budget forecasting view |
| `daily-routine.vue` | `views/` | Daily routine management |
| `spraying_routine.vue` | `views/` | Spraying schedule management |
| `action-planner.vue` | `views/` | MyTask + Annual Plan |

### 2.2 Fertilisation Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `FertilisationPlanner.vue` | `components/fertilisation_planner/` | Weekly planning grid |
| `ProductComposition.vue` | `components/fertilisation_planner/` | NPK editor |
| `TemplatesList.vue` | `components/fertilisation_planner/templates/` | Template management |
| `Overview.vue` | `components/fertilisation_planner/` | Summary view |

### 2.3 Budget Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `ProductPlannerTable.vue` | `components/budget_planner/` | Product budget table |
| `CompositionIndex.vue` | `components/budget_planner/composition/substances/` | Substance breakdown |

### 2.4 Action Planner Components

| Component | Path | Responsibility |
|:---|:---|:---|
| `ActionTemplate.vue` | `components/action-planner/` | Template selector |
| `SiteChooseArea.vue` | `components/action-planner/` | Area selection (Golf) |
| `SiteChoosePitches.vue` | `components/action-planner/` | Pitch selection (Football) |
| `MyTask.vue` | `components/action-planner/` | Task list view |
| `AnnualPlan.vue` | `components/action-planner/` | Annual timeline |

---

## 3. State Management

### 3.1 Fertilisation Planner Store

**File**: `web/src/store/fertilisationplanner.js`

```javascript
// State Structure (partial)
{
  tenantId: null,
  productLists: [],
  activePlanId: null,
  isEdited: false,
  isOpenConfirmModal: false,
  fertilisationPlanFilters: {
    year: new Date().getFullYear()
  },
  compositionValues: {
    N: {}, P: {}, K: {}, Ca: {}, Fe: {},
    S: {}, Zn: {}, Mg: {}, Cu: {}, Mn: {}, B: {}
  },
  templateLists: [],
  isFertilisationChangedValue: null,
  isFertilisationChangedYear: null
}
```

### 3.2 Key Store Actions (20+ total)

| Action | Description | API Call |
|:---|:---|:---|
| `getProductForFertilize()` | Load products | GET |
| `getFertilisationPlansPerTenant()` | Load plans | GET |
| `getTemplateDataForPreview()` | Print preview data | GET |
| `getFertilizationTemplateData()` | Template details | GET |
| `newFPTemplate(data)` | Create template | POST |
| `addFertilization(data)` | Add fertilization | POST |
| `updateFPTemplate(data)` | Update NPK | PATCH |
| `updateProductData(id, data)` | Update product | PATCH |
| `updateProductComposition(data)` | Update composition | PATCH |
| `deleteProductTemplate(id)` | Delete template | DELETE |
| `getFertilisationTemplateLists()` | List templates | GET |
| `getFertilisationYears(id, checked)` | Years for plan | GET |
| `createTemplateWithData(data)` | Create with data | POST |
| `storeNPKValues(data)` | Save NPK values | POST |
| `getFertilizationTemplateDataById(id, year)` | By ID | GET |
| `getSavedNPKDataForTemplate(id, year)` | NPK data | GET |

### 3.3 Budget Planner Store

**File**: `web/src/store/budgetplanner.js`

| Action | Description |
|:---|:---|
| `getBudgetPlanData(year)` | Load budget data |
| `getProductForFertilize()` | Load products |
| `getFertilisationPlansPerTenant()` | Load plans |
| `getSuppliersList()` | Load suppliers |
| `getFertilizationTemplateData(plans, year)` | Template data |

### 3.4 Action Planner Store

**File**: `web/src/store/actionplanner/action-planner.js`

| Action | Description |
|:---|:---|
| `getTemplateLists(year)` | Load templates by year |
| `getActionByID(id)` | Get action details |
| `getActionPlanLists(id)` | Get plan lists |
| `createActionPlanner(data)` | Save action plan |

---

## 4. Data Models

### 4.1 Fertilization Plan Template

**Table**: `fertilization_plan_template`

| Column | Type | Description |
|:---|:---|:---|
| `idfertilization_plan_template` | CHAR(36) | UUID identifier |
| `idtenant` | CHAR(36) | Tenant reference |
| `name` | VARCHAR(100) | Template name |
| `year` | YEAR | Planning year |
| `created_at` | TIMESTAMP | Creation timestamp |

### 4.2 Fertilization Plan

**Table**: `fertilization_plan`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `template_id` | CHAR(36) | Template reference |
| `idproduct` | CHAR(36) | Product reference |
| `week` | INT | Week number (1-52) |
| `month` | INT | Month number |
| `quantity` | DECIMAL(12,4) | Application quantity |

### 4.3 Product Composition

**Table**: `product_composition`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `idproduct` | CHAR(36) | Product reference |
| `nutrient` | VARCHAR(10) | N, P, K, etc. |
| `percentage` | DECIMAL(5,2) | Nutrient percentage |

### 4.4 Daily Routine

**Table**: `daily_routine`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `idtenant` | CHAR(36) | Tenant reference |
| `name` | VARCHAR(100) | Routine name |
| `site_ids` | JSON | Array of site IDs |
| `task_count` | INT | Number of tasks |
| `updated_at` | TIMESTAMP | Last update |

### 4.5 Action Plan

**Table**: `action_plan`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `template_id` | CHAR(36) | Template reference |
| `site_ids` | JSON | Site references |
| `choose_area_ids` | JSON | Area references |
| `year` | YEAR | Planning year |
| `task_lists` | JSON | Task configurations |
| `annual_plan` | JSON | Month/week data |

---

## 5. API Specification

### 5.1 Get Fertilisation Template Data

```
GET /api/v2/fertilization-template-data?idtenant={tenant}&plan_id={id}&year={year}
Authorization: Bearer {token}

Response: 200 OK
{
  "template_id": "template-uuid",
  "products": [
    {
      "idproduct": "product-uuid",
      "name": "NPK 15-0-15",
      "weeks": {
        "1": 50, "2": 0, "3": 75, ...
      }
    }
  ],
  "npk_targets": {
    "N": { "target": 200, "actual": 185 },
    "P": { "target": 50, "actual": 45 },
    "K": { "target": 150, "actual": 140 }
  }
}
```

### 5.2 Create Action Planner

```
POST /api/v2/action-planner
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "template_id": "template-uuid",
  "site_ids": ["site-uuid-1"],
  "choose_area_ids": ["area-uuid-1", "area-uuid-2"],
  "task_lists": [
    { "task_id": "task-uuid", "count": 5, "name": "Bunker Repair" }
  ],
  "annualPlan": {
    "1": { "wk1": true, "wk2": false, ... },
    "2": { ... }
  },
  "type": "MyTask"
}

Response: 201 Created
{
  "id": "action-plan-uuid",
  "message": "Action plan saved"
}
```

### 5.3 Get Budget Plan Data

```
GET /api/v2/budget-plan-data?idtenant={tenant}&year={year}
Authorization: Bearer {token}

Response: 200 OK
{
  "products": {
    "product-uuid": {
      "name": "Fungicide X",
      "current_stock": 25,
      "planned_quantity": 100,
      "unit_price": 45.00,
      "budget_plans": { ... }
    }
  }
}
```

---

## 6. Sequence Diagrams

### 6.1 Fertilisation Planning Flow

```
┌──────┐    ┌────────────┐    ┌─────────┐    ┌─────────┐
│ User │    │ FertPlan   │    │  Store  │    │   API   │
└──┬───┘    └─────┬──────┘    └────┬────┘    └────┬────┘
   │  Select Template │              │              │
   │─────────────────>│              │              │
   │                  │ getTemplateData()           │
   │                  │─────────────>│              │
   │                  │              │ GET template │
   │                  │              │─────────────>│
   │                  │              │  Data        │
   │                  │              │<─────────────│
   │                  │  Update Grid │              │
   │                  │<─────────────│              │
   │  Edit Week Value │              │              │
   │─────────────────>│              │              │
   │                  │ updateFPTemplate()          │
   │                  │─────────────>│              │
   │                  │              │ PATCH        │
   │                  │              │─────────────>│
```

---

## 7. Business Logic

### 7.1 NPK Calculation

```javascript
// Calculate actual NPK from scheduled products
function calculateActualNPK(scheduledProducts, compositions) {
  const totals = { N: 0, P: 0, K: 0 };
  
  scheduledProducts.forEach(product => {
    const comp = compositions[product.idproduct];
    const qty = product.totalQuantity;
    
    totals.N += (qty * comp.N / 100);
    totals.P += (qty * comp.P / 100);
    totals.K += (qty * comp.K / 100);
  });
  
  return totals;
}
```

### 7.2 Budget Calculation

```javascript
// Calculate purchase requirements
function calculatePurchaseNeed(planned, currentStock, unitSize) {
  const rawNeed = planned - currentStock;
  const unitsNeeded = Math.ceil(rawNeed / unitSize);
  return Math.max(0, unitsNeeded);
}
```

---

## 8. Integration Points

| System | Type | Data Flow |
|:---|:---|:---|
| Inventory | Reference | Stock levels, products |
| Schedule | Consumer | Routine loading |
| Agronomy | Reference | Nutrient recommendations |
| Tasks | Reference | Activity definitions |

---

## 9. Performance Requirements

| Metric | Requirement |
|:---|:---|
| Template load | < 3 seconds |
| Budget calculation | < 2 seconds |
| Routine save | < 1 second |
| Annual plan render | < 2 seconds |

---

**Document End**
