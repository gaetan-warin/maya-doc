# Planner Module
## Functional Requirements Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-FRS-PLNR-001 |
| **Version** | 1.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Product Team |
| **Classification** | Internal |

---

## Executive Summary

The **Planner Module** provides comprehensive planning tools for agronomic operations, financial budgeting, and daily task scheduling. It encompasses five integrated sub-modules: **Fertilisation Planner** (nutrient scheduling), **Budget Planner** (product forecasting), **Daily Routine** (recurring task templates), **Spraying Routine** (chemical application scheduling), and **Action Planner** (long-term action tracking).

This document defines the functional requirements for nutrient management, budget calculation, task templating, and strategic planning. It integrates with Inventory, Tasks, and Agronomy modules to provide unified operational planning.

**Key Business Outcomes:**
- **Nutrient Optimization**: Precise NPK planning reduces waste and optimizes turf health.
- **Budget Control**: Forecast product needs and costs before purchasing.
- **Operational Efficiency**: Reusable templates reduce daily planning overhead.
- **Strategic Visibility**: Annual views enable long-term resource planning.

---

## 1. Scope & Objectives

### 1.1 In Scope
- Fertilisation program planning with NPK targets
- Product budget forecasting and cost calculation
- Daily task routine templating
- Spraying schedule planning
- Annual action planning and tracking

### 1.2 Out of Scope
- Automated task execution (templates feed the Schedule)
- Weather-based spray window recommendations
- Integration with external agronomic consultants

### 1.3 Target Users
| Role | Primary Use Case |
|:---|:---|
| **Agronomist** | Fertilisation and spraying planning |
| **Superintendent** | Daily routine management |
| **Finance Manager** | Budget planning and forecasting |
| **Operations Manager** | Annual action oversight |

---

## 2. Fertilisation Planner

### 2.1 Overview

The Fertilisation Planner enables precise nutritional program management with product composition tracking and weekly application scheduling.

### 2.2 Tabs

| Tab | Purpose |
|:---|:---|
| **Product Composition** | Manage nutrient percentages per product |
| **Fertilisation Planning** | Schedule applications by week |

### 2.3 Product Composition

Manage nutrient breakdown for each fertilisation product:

| Nutrient | Description |
|:---|:---|
| N (Nitrogen) | Primary growth nutrient |
| P (Phosphorus) | Root development |
| K (Potassium) | Disease resistance |
| Ca (Calcium) | Cell wall strength |
| Fe (Iron) | Chlorophyll production |
| S (Sulfur) | Protein synthesis |
| Zn (Zinc) | Enzyme function |
| Mg (Magnesium) | Photosynthesis |
| Cu (Copper) | Lignin formation |
| Mn (Manganese) | Enzyme activation |
| B (Boron) | Cell division |

### 2.4 Fertilisation Planning

Weekly scheduling grid with:
- Year selection (2022-2026)
- Site and area filtering (Greens, Fairways, Tees, etc.)
- Monthly view with WK1-WK5 columns
- Target vs. Actual nutrient comparison
- Template-based planning

### 2.5 Controls

| Control | Description |
|:---|:---|
| Site Selection | Multi-select for sites |
| Area Selection | Filter by zone type |
| Year | Year filter |
| Spraying Routine | Quick link to spraying |
| Print | Export planning table |

---

## 3. Budget Planner

### 3.1 Overview

The Budget Planner forecasts product requirements and calculates costs based on fertilisation plans.

### 3.2 Data Table Columns

| Column | Description |
|:---|:---|
| Product Name | Fertiliser/chemical name |
| Supplier | Dropdown for supplier selection |
| Purchase Unit Price | Cost per unit |
| Unit Quantity | Package size |
| Current Stock | Available inventory (with out-of-stock warning) |
| Planned Total Quantity | Required for year |
| Quantity to Purchase | Gap between stock and planned |
| Total Cost | Calculated purchase cost |
| Actual Order | Confirmed order quantity |

### 3.3 Template Filters

Quick filter templates:
- Fairways
- Greens
- All templates checkbox list

### 3.4 Year Selection

Filter budget data by year (2022 onwards).

### 3.5 Calculations

```
Quantity to Purchase = Planned Total Quantity - Current Stock
Total Cost = Quantity to Purchase × Purchase Unit Price
```

---

## 4. Daily Routine

### 4.1 Overview

Daily Routine provides reusable task templates that can be loaded into the Schedule to streamline recurring operations.

### 4.2 Routine List

| Column | Description |
|:---|:---|
| Routine Name | Template name |
| Sites | Associated locations |
| Number of Tasks | Task count in routine |
| Last Modified | Update timestamp |

### 4.3 Routine Task Configuration

Each task within a routine specifies:

| Field | Description |
|:---|:---|
| Staff | Assigned personnel |
| Site | Location |
| Hole | Specific hole (Golf) |
| Machinery | Equipment to use |
| Area | Zone type |
| Shift | AM / PM / Overtime |
| Duration | Time allocation |

### 4.4 Loading Routines

Routines are designed to be loaded into the Schedule page, populating daily tasks with predefined configurations.

---

## 5. Spraying Routine

### 5.1 Overview

Spraying Routine plans chemical and fertiliser application schedules with precise product dosing and area targeting.

### 5.2 Routine List

| Column | Description |
|:---|:---|
| Routine Name | Template name |
| Target Areas | Zones to spray |
| Stock Items | Products used (with dosage) |
| Timestamp | Last update |

### 5.3 Spraying Form Fields

| Field | Description |
|:---|:---|
| Time Window | Start and end time |
| Site | Location selection |
| Hole | Hole multi-select (Golf) |
| Machinery | Sprayer/equipment |
| Area Grid | Interactive zone selector (Infrastructure, Fairways, Tee, etc.) |

### 5.4 Product Details

For each product in the routine:
- Dosage (kg/ha or L/ha)
- Total quantity calculation
- Stock level reference

---

## 6. Action Planner

### 6.1 Overview

Action Planner provides strategic, long-term task planning with annual visualizations and zone-based organization.

### 6.2 Views

| View | Description |
|:---|:---|
| **My Tasks** | List-based task selection |
| **Annual Plan** | 12-month timeline with weekly columns |

### 6.3 My Tasks View

| Feature | Description |
|:---|:---|
| Task Toggles | Enable/disable activities |
| Task List | Bunker Reparation, Blow Leaves, etc. |
| Count per Task | Occurrence tracking |

### 6.4 Annual Plan View

Monthly grid showing:
- 12 months (Jan-Dec)
- WK1-WK5 columns per month
- Action status indicators
- Zone filtering

### 6.5 Controls

| Control | Description |
|:---|:---|
| Zone Selection | Filter by zone (e.g., "Zon 2") |
| Template Selection | Choose action template |
| Year | Year filter |
| Site/Area | Location filtering |
| Save & Refresh | Persist changes |

---

## 7. Integration Points

| System | Integration Type | Data Flow |
|:---|:---|:---|
| **Inventory Module** | Reference | Stock levels, product data |
| **Schedule Module** | Consumer | Routine loading |
| **Agronomy Module** | Reference | Nutrient recommendations |
| **Task Module** | Reference | Activity definitions |

---

## 8. Acceptance Criteria

1. Fertilisation Planner displays weekly grid with NPK targets.
2. Product composition can be edited and saved.
3. Budget Planner calculates required purchase quantities.
4. Daily routines can be created with multiple tasks.
5. Spraying routines specify products with dosage calculations.
6. Action Planner shows annual timeline with weekly granularity.
7. All planners filter by site, area, and year.

---

**Document End**
