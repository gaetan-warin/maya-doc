# Maya Unified Task System — Functional Requirements Specification (FRS)

> **Version:** 1.0  
> **Generated:** 2025-12-22  
> **Source:** Comprehensive extraction from 13 Notion documentation pages

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Business Context](#2-business-context)
3. [Functional Domains](#3-functional-domains)
   - 3.1 Task Management
   - 3.2 Nutritional Inputs (Spraying/Fertilisation)
   - 3.3 Labour & Time Tracking
   - 3.4 Fleet Management
   - 3.5 Weather & Environmental Data
   - 3.6 Reporting & Analytics
4. [Use Cases](#4-use-cases)
5. [API Contracts](#5-api-contracts)
6. [Business Rules](#6-business-rules)
7. [Data Calculations](#7-data-calculations)
8. [Known Issues & Requirements](#8-known-issues--requirements)
9. [Project Roadmap](#9-project-roadmap)
10. [Source Documents Index](#10-source-documents-index)

---

## 1. Executive Summary

Maya is a platform used daily in **golf courses**, **football stadiums**, and **multisport venues** to plan, execute, and analyse operational work. The Schedule module is the most business-critical component, managing:

- **Tasks** — All operational activities (mowing, aeration, repairs, etc.)
- **Nutritional Inputs** — Fertilisation, spraying, chemical applications
- **Labour** — Staff assignments, hours, overtime, TOIL
- **Fleet** — Machinery, maintenance, compliance
- **Reporting** — Analytics, cost analysis, compliance tracking

### Current State Problems

| Issue | Impact |
|-------|--------|
| 20+ API calls per page load | Poor performance |
| Fragmented data models | Inconsistent FE/BE contracts |
| Tasks/Spraying in separate tables | Duplicate logic, reporting errors |
| `merge_key` anti-pattern | Data duplication, complex queries |
| Limited compliance tracking | No operator-machine linkage |

### Solution Direction

A **unified task architecture** with:
- Single `tasks` table as the core entity
- Extension tables for domain-specific data (nutritional, leave)
- Junction tables for many-to-many relationships
- Phased migration preserving backward compatibility

---

## 2. Business Context

### Supported Business Types

| Type | Entities | Surface Logic |
|------|----------|---------------|
| **GOLF** | Courses → Holes → Areas (Greens, Fairways, Tees) | API-based surface calculation |
| **FOOTBALLPITCH** | Sites → Pitches (sub-sites) → Zones | Local pitch aggregation |
| **GRASS_ROOT** | Multi-sport venues | Hybrid |

### Core Concepts

**A task = planned work + actual work**, carried out by specific staff, machines, and parameters, across one or multiple locations.

Includes:
- Mowing, fertilising, spraying, topdressing, bunker raking
- Hand-watering, line marking, grooming artificial pitches
- Aeration, verticutting, repairs, renovations
- Irrigation checks, inspections
- All nutritional/chemical/amendment applications

---

## 3. Functional Domains

### 3.1 Task Management

#### Task Status Flow

```
todo → in_progress → completed
         ↓
     not_completed
```

| Status | `running` | `task_start` | `task_end` |
|--------|-----------|--------------|------------|
| `todo` | 0 | null | null |
| `in_progress` | 0 | value | null |
| `completed` | 1 | value | value |
| `not_completed` | 0 | value | value |

#### Task Types

| Type | `schedule_task_type` | Description |
|------|---------------------|-------------|
| Site-based | `site-based` | Works across holes/areas within a site |
| Tenant-based | `tenant-based` | Works across entire tenant scope |

#### Multi-Site/Multi-Location Tasks

> **Critical Rule:** A staff member logs ONE work session, not multiple.

When a task covers multiple locations, duration splits evenly:
- **Example:** Topdressing Greens 1, 2, 3 with 1 staff for 3 hours = **1h per green**

#### Planned Duration Logic

The planned duration = **expected time per staff member**, NOT total labour.

**Labour Calculation:**
```
Planned Labour = Duration × Number of Staff
```

**Example:**
- Aeration of 5 greens (8am → 11am = 3h), 2 staff
- Planned labour = 3h × 2 = **6 labour hours**

---

### 3.2 Nutritional Inputs (Spraying/Fertilisation)

Nutritional inputs are **tasks with additional parameters**.

#### Three Operational Modes

| Mode | Features | Use Case |
|------|----------|----------|
| **Standard** | Product qty/ha, Total qty | Basic applications |
| **Advanced Spraying** | + Flow rate, Speed, RPM, Tank size, Dissolution volumes | Precision application |
| **Precision Management** | + Nozzle type, Nozzle spacing, Live N units | Highest accuracy |

#### Advanced Mode Calculations

Given user inputs:
- Flow Rate (LPM)
- Tractor Speed (KM/H)
- RPM
- Dissolution Volume per Hectare
- Tank Size

Maya calculates:
- **Total Dissolution Volume** for entire area
- **Number of Tanks** needed
- **Recommended Flow Rate** for nozzle selection
- **Nitrogen Unit Quantity** per product

#### Stock Logic

| Item Type | Consumed By |
|-----------|-------------|
| Product | Nutritional inputs |
| Non-product | Regular tasks |

---

### 3.3 Labour & Time Tracking

#### Time Slots

| Slot | Description |
|------|-------------|
| AM | Morning shift |
| PM | Afternoon shift |
| Overtime | Beyond scheduled hours |

#### Overtime Rules (TO-BE Logic)

**Type 1:** Task explicitly flagged as overtime
**Type 2:** Staff member exceeds daily scheduled hours

#### TOIL (Time Off in Lieu)

**Two Methods for Recording:**

1. **From Team Management Page:**
   - Click "+" button → Select TOIL or Overtime
   - Enter staff, date, start/end time, comments
   - Saved to member profile

2. **From Schedule Page:**
   - Click TOIL icon for quick entry
   - Click Overtime button (turns red when logged)

#### Leave Management

Separate extension table (`task_ext_leave`) for:
- Leave type tracking
- Approval workflow
- Hours/days calculation

---

### 3.4 Fleet Management

#### Machine Addition Process

1. Load brands, models, serials, fuel types via `getMachineData`
2. Model selection determines category and fuel type
3. Two-step save:
   - Save basic machine data → `Add machine` API
   - Add machine time → `Add machine time` API
4. Machine name format: `[Model Name]_[Machine Number]`

#### Maintenance Process

Captures:
- Maintenance name
- Machine selection
- Status: `urgent | minor | done | upcoming`
- Date, Assigned User, Created User
- Cost, Hours in Use

Data stored in `machine_maintenance` and `machine_time` tables.

#### Recurring Maintenance

1. Call recurrence API first
2. Then maintenance API
3. Toggle enable/disable
4. Deactivate maintenance option

#### Incident Reporting

- POST/PATCH API captures: severity, status, comments
- Links to machines, tasks, or spraying sessions
- Triggers notifications (Email/WhatsApp) via `notifyIncident`

---

### 3.5 Weather & Environmental Data

#### Available Metrics (14 Total)

| Code | Metric | Unit |
|------|--------|------|
| `air_temperature` | Air temperature | °C |
| `air_humidity` | Air humidity | % |
| `wind_speed` | Wind speed | m/s |
| `wind_gust` | Wind gust | m/s |
| `radiation` | Solar radiation | W/m² |
| `rainfall` | Precipitation | mm |
| `et24` | 24-hour evapotranspiration | mm/24h |
| `et` | Evapotranspiration | mm |
| `leaf_wetness` | Leaf wetness | % |
| `soil_temperature` | Soil temp (5cm) | °C |
| `soil_humidity` | Soil humidity (5cm) | % |
| `soil_oxygen` | Soil Oxygen | % |
| `soil_salinity` | Soil salinity | dS/m |
| `dli` | Daily Light Integral | mol/m²/day |

#### Weather API

**Endpoint:** `GET https://api2.mayaglobal.io/api/v4/metrics`

**Parameters:**
- `codes` (required): Comma-separated metric keys
- `idsite` (optional): Site UUID filter

**Authentication:** Bearer token from Maya settings

---

### 3.6 Reporting & Analytics

#### Summary Tables Architecture

**Event-Driven Flow:**
```
User Action → Repository → Event Dispatch → Listener → Service → Database
```

**Record Types:**
- `task_schedule` — Regular tasks
- `task_spraying` — Nutritional inputs
- `leave` — Leave entries
- `toil` — Time off in lieu
- `overtime` — Extra hours

#### Key Aggregation Fields

- `work_hours` — Total work hours
- `estimated_time` — Planned duration
- `total_resource_cost` — Cost calculation
- `active_substance` — JSON for nutritional tracking
- `product_substance` — JSON for product details

---

## 4. Use Cases

### UC-001: Create Daily Schedule Task

**Actor:** Greenkeeper / Groundsman

**Preconditions:** User logged in, has schedule permissions

**Flow:**
1. Navigate to Schedule page → Day Plan tab
2. Click "Add Task"
3. Select:
   - Action type (Mowing, Aeration, etc.)
   - Sites/Holes/Zones
   - Staff members
   - Machinery
   - Time slot (AM/PM/Overtime)
   - Duration
4. System generates `merge_key` (legacy) or `task_id` (TO-BE)
5. Task appears in calendar and staff assignments

**Postconditions:** Task created, staff notified (if enabled)

---

### UC-002: Record Spraying Application

**Actor:** Spray Operator

**Preconditions:** Products in stock, weather within parameters

**Flow:**
1. Select Nutritional Input mode (Standard/Advanced/Precision)
2. Choose product(s) from stock
3. Enter application rate (kg/ha or L/ha)
4. Select target areas (holes, zones)
5. System calculates:
   - Total quantity needed
   - Number of tanks (Advanced mode)
   - Nitrogen units applied
6. Record actual application with:
   - Weather conditions at time
   - Operator
   - Machine used
7. Stock automatically decremented

**Postconditions:** Spraying logged, stock updated, compliance record created

---

### UC-003: Log Overtime Hours

**Actor:** Team Manager

**Method 1 — From Schedule:**
1. Click Overtime button on task
2. Icon turns red (confirmed)
3. Entry saved to staff record

**Method 2 — From Team Management:**
1. Click "+" button
2. Select "Overtime"
3. Enter staff, date, times, comments
4. Save

**Postconditions:** Overtime recorded, visible in Hours view

---

### UC-004: Multi-Site Task Distribution

**Actor:** Course Manager

**Scenario:** Topdressing 3 greens with 1 staff member

**Flow:**
1. Create task: "Topdressing" for Greens 1, 2, 3
2. Mobile duration: 3 hours
3. System distributes: 1h per green
4. Each location receives proportional labour allocation

**Rule:** Staff logs ONE session; system handles distribution

---

## 5. API Contracts

### Schedule Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v2/createSchedule` | Create new task |
| GET | `/api/v2/getScheduleIntialData` | Page load data |
| GET | `/api/v2/schedules` | Tasks + Sprayings for calendar |
| PUT | `/api/v2/update-schedule` | Update by merge_key |
| DELETE | `/schedule/{mergeKey}` | Delete entire schedule group |
| DELETE | `/schedule/task/{taskId}` | Delete single task |

### Product Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/tenant/{idtenant}/products` | List products |
| POST | `/tasks/products` | Add products to task |
| PUT | `/tasks/products` | Edit task products |
| GET | `/api/v2/action-tasks-by-date/{idtenant}/{date}` | Tasks by date |

### Weather API

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v4/metrics?codes={codes}&idsite={idsite}` | Weather metrics |

---

## 6. Business Rules

### BR-001: Planned Labour Calculation
```
Planned Labour Hours = Task Duration × Number of Assigned Staff
```

### BR-002: Multi-Location Distribution
```
Hours per Location = Total Duration / Number of Locations
```

### BR-003: Surface Area Normalisation
```
Normalised Surface (ha) = Calculated Surface / 10000
```

### BR-004: Overtime Determination
```
IF task.is_overtime = true
  → Record as overtime
ELSE IF actual_hours > staff.scheduled_hours
  → Excess hours recorded as overtime
```

### BR-005: Stock Consumption
```
IF record_type = 'task_spraying'
  → Consume stock of type 'Product'
ELSE
  → Consume stock of type 'Non-product'
```

### BR-006: Pro-Rata Quantity Distribution
```
Zone Quantity = (Zone Surface / Total Surface) × Total Quantity Applied
```

---

## 7. Data Calculations

### Surface Area (GOLF)

**With Holes Selected:**
```javascript
totalSurface = await getAreaHolesSurface(site, holes, areas)
return totalSurface.data.total_surface / 10000
```

**Without Holes:**
```javascript
totalSurface = calculateTotalSurface(sites, selectedAreas)
return totalSurface / 10000
```

### Surface Area (FOOTBALLPITCH)

**With Holes Selected:**
```javascript
filteredPitches = filterPitchesBySiteAndHoles(site, holes)
totalZones = site.areas.length

if (selectedAreas.length > 0) {
  // Distribute per zone, multiply by selected areas
  for (pitch in filteredPitches) {
    surface += (pitch.surface / totalZones) * selectedAreas.length
  }
} else {
  // Sum all filtered pitches
  surface = sum(filteredPitches.surface)
}
return surface / 10000
```

### Spraying Calculations (Advanced Mode)

```javascript
// User inputs
flowRate_LPM = input
tractorSpeed_KMH = input
rpm = input
dissolutionVolume_perHa = input
tankSize_L = input

// Calculated outputs
totalDissolutionVolume = dissolutionVolume_perHa * totalSurface_ha
numberOfTanks = ceil(totalDissolutionVolume / tankSize_L)
recommendedFlowRate = calculateNozzleFlowRate(flowRate_LPM, tractorSpeed_KMH)
nitrogenUnits = calculateNitrogenFromProducts(products, quantities)
```

---

## 8. Known Issues & Requirements

### Reporting Bugs (Priority)

| Issue | Section | Fix Required |
|-------|---------|--------------|
| Triplicate rows | List of Treatments | De-duplicate query |
| Missing NPK values | Treatments | Query returns empty if no substance |
| Unit mismatch | Quantity Timeline | Standardise kg/ha vs g/sqm |
| Wrong pricing | Cost Analysis | Fix price × quantity calculation |
| Missing currency | Cost displays | Add currency field |
| Random colors | Charts | Implement color palette |

### Pro-Rata Calculation Fix

**Current:** Shows raw quantity regardless of zone
**Required:** `quantity × zone_surface_area`

**Example:**
- Greens = 1.2 ha, Fairways = 16 ha
- Quantity = 12 kg/ha
- Greens total = 12 × 1.2 = 14.4 kg
- Fairways total = 12 × 16 = 192 kg

---

## 9. Project Roadmap

| Phase | Timeline | Owner | Deliverable |
|-------|----------|-------|-------------|
| 1A | Dec 16 – Jan 5 | Simon | API Contract Specification |
| 1B | Dec 16 – Dec 31 | Gaetan | DB Schema Analysis & TO-BE Proposal |
| 2 | January 2025 | Simon + Dilanka | New Vue.js + Optimised Laravel |
| 3 | February 2025 | Simon + Dilanka | FE-BE Integration & QA |
| 4 | Q1–Q2 2025 | TBD | Database Schema Migration |

**Target Completion:** Summer 2026

### Migration Strategy

1. **Parallel tables** or compatibility layer
2. **Gradual switching** by feature/context
3. **No "big bang"** migration
4. Backward compatibility preserved throughout

---

## 10. Source Documents Index

All source documentation archived in [`docu/notion/`](file:///c:/www/mayaApp/docu/notion/):

| Document | Content |
|----------|---------|
| [task_labour_refactoring.md](file:///c:/www/mayaApp/docu/notion/task_labour_refactoring.md) | TO-BE model, core concepts, overtime rules |
| [day_plan_task_management.md](file:///c:/www/mayaApp/docu/notion/day_plan_task_management.md) | Task status flow, creation process |
| [task_parameter.md](file:///c:/www/mayaApp/docu/notion/task_parameter.md) | Product linking APIs |
| [schedule_api.md](file:///c:/www/mayaApp/docu/notion/schedule_api.md) | All schedule endpoints |
| [nutritional_inputs.md](file:///c:/www/mayaApp/docu/notion/nutritional_inputs.md) | Three operational modes |
| [surface_area_calculations.md](file:///c:/www/mayaApp/docu/notion/surface_area_calculations.md) | Golf/Football surface logic |
| [weather_api.md](file:///c:/www/mayaApp/docu/notion/weather_api.md) | 14 weather metrics |
| [summary_tables.md](file:///c:/www/mayaApp/docu/notion/summary_tables.md) | Database schema, event architecture |
| [machines_and_staff.md](file:///c:/www/mayaApp/docu/notion/machines_and_staff.md) | Fleet management workflows |
| [manage_toil_overtime.md](file:///c:/www/mayaApp/docu/notion/manage_toil_overtime.md) | TOIL/Overtime recording |
| [interactive_reports_nutritional.md](file:///c:/www/mayaApp/docu/notion/interactive_reports_nutritional.md) | Reporting bugs & requirements |
| [schedule_page_revamp.md](file:///c:/www/mayaApp/docu/notion/schedule_page_revamp.md) | Phased execution plan |
| [schedule_improvement.md](file:///c:/www/mayaApp/docu/notion/schedule_improvement.md) | Epic resources |

---

*This FRS consolidates all functional requirements from the Maya Notion workspace. For technical implementation details, see [design_proposal_unified_tasks_tobe.md](file:///c:/www/mayaApp/docu/design_proposal_unified_tasks_tobe.md).*
