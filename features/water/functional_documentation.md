# Water & Irrigation Management
## Functional Requirements Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-FRS-WATER-001 |
| **Version** | 3.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Product Team |
| **Classification** | Internal |

---

## Executive Summary

The **Water & Irrigation Management** module provides comprehensive tracking of water consumption, irrigation activities, and moisture balance across managed turf properties. It adheres to Maya's Unified Task Model, ensuring that irrigation labor is tracked consistently with all other operational activities.

This document defines the functional requirements for water source management, consumption tracking, and the critical Water Balance metric that drives irrigation decision-making.

**Key Business Outcomes:**
- **Regulatory Compliance**: Monthly and annual consumption tracking ensures adherence to extraction permits.
- **Cost Control**: Visibility into water usage enables budget management and leak detection.
- **Agronomic Optimization**: Water Balance metric prevents both drought stress and over-watering.

---

## 1. Scope & Objectives

### 1.1 In Scope
- Water source configuration and management
- Consumption recording (meter readings, manual entry)
- Water Balance calculation (Rainfall vs. Evapotranspiration)
- Consumption analytics and reporting
- Irrigation task scheduling and labor tracking

### 1.2 Out of Scope
- Automated irrigation control commands (handled by external controllers)
- Pump station maintenance scheduling

### 1.3 Target Users
| Role | Primary Use Case |
|:---|:---|
| **Irrigation Technician** | Daily readings, system monitoring |
| **Superintendent** | Water Balance review, irrigation decisions |
| **Operations Manager** | Consumption vs. budget analysis |
| **Compliance Officer** | Extraction permit reporting |

---

## 2. Water Source Management

### 2.1 Source Types

The system supports multiple water input methods:

| Source Type | Data Entry Method | Example |
|:---|:---|:---|
| **Meter Reading** | Cumulative counter value | Municipal supply meter |
| **Daily Volume** | Direct volume entry | Pump station runtime × flow rate |
| **Calculation** | Derived from runtime | Lake pump with known GPM |

### 2.2 Source Configuration

| Attribute | Description |
|:---|:---|
| Name | User-defined identifier |
| Type | Meter / Pump / Manual |
| Unit | Cubic meters (m³) or Gallons |
| Is Legacy | Boolean flag for retired sources |

### 2.3 Legacy Source Handling

When a water source is replaced (e.g., meter upgrade), the old source is marked as **Legacy**:
- Legacy sources are excluded from new entry selection
- Legacy data remains in historical calculations
- This ensures accurate year-over-year consumption comparisons

---

## 3. Water Balance

### 3.1 Definition

The Water Balance is the fundamental agronomic metric for irrigation decision-making. It represents the net moisture change over a 24-hour period.

**Formula:**
```
Water Balance = Rainfall (24h) - Evapotranspiration (24h)
```

### 3.2 Data Sources

| Metric | Source | Code |
|:---|:---|:---|
| Rainfall (24h) | Weather station rain gauge | `LAST_24H_RAINFALL` |
| Evapotranspiration (24h) | Calculated via Penman-Monteith | `ETP24` |

### 3.3 Interpretation

| Balance | Meaning | Recommended Action |
|:---|:---|:---|
| **Positive (+)** | Moisture surplus | Reduce or skip irrigation |
| **Zero (0)** | Equilibrium | Monitor soil conditions |
| **Negative (−)** | Moisture deficit | Irrigation required |

### 3.4 Example

| Metric | Value |
|:---|:---|
| Rainfall (24h) | 3.2 mm |
| ET (24h) | 5.8 mm |
| **Water Balance** | **−2.6 mm** (deficit) |

**Decision**: Irrigation is recommended to replace the 2.6 mm moisture deficit.

---

## 4. Consumption Tracking

### 4.1 Reading Entry

Users enter consumption data via the Water form:
- Select water source
- Enter date
- Enter reading (meter value or volume)
- Add optional notes

### 4.2 Consumption Calculation

For meter-based sources, consumption is calculated as:
```
Daily Consumption = Current Reading − Previous Reading
```

### 4.3 Aggregation Levels

| Period | Display |
|:---|:---|
| Daily | Individual readings |
| Monthly | Sum of daily consumption |
| Yearly | Sum of monthly consumption |
| Days Watered | Count of days with consumption > 0 |

---

## 5. Irrigation as Unified Task

### 5.1 Integration with Task Module

Irrigation activities (hand watering, system checks, repairs) follow the same Unified Task Model as all other operations:

| Attribute | Irrigation Example |
|:---|:---|
| Planned Duration | 2 hours per staff |
| Assigned Staff | 3 irrigators |
| Total Labor | 6 labor hours |
| Overtime | Applies if work exceeds schedule |

### 5.2 Consumption vs. Labor

These are distinct but related metrics:

| Metric | Measures | High Value Indicates |
|:---|:---|:---|
| **Consumption** | Volume of water used (m³) | Automated/efficient system |
| **Labor** | Time spent managing irrigation | Manual intervention required |

**Relationship**: Properties with modern automated irrigation typically show high consumption but low labor. Properties relying on hand-watering show lower consumption but higher labor.

---

## 6. Reporting & Analytics

### 6.1 Dashboard KPIs

| KPI | Description |
|:---|:---|
| Water Balance | Current 24h net moisture |
| Monthly Consumption | m³ used in current month |
| Yearly Consumption | m³ used year-to-date |
| Days Watered | Number of irrigation days this month |
| Average Daily | Mean consumption per day |

### 6.2 Trend Analysis

Year-over-year consumption comparison enables:
- Identification of usage anomalies (potential leaks)
- Verification of efficiency improvement initiatives
- Permit compliance forecasting

---

## 7. Integration Points

| System | Integration Type | Data Flow |
|:---|:---|:---|
| **Weather Module** | Inbound | ET and Rainfall for Water Balance |
| **Task Module** | Bidirectional | Irrigation tasks appear in schedule |
| **Map Module** | Reference | Site filtering for consumption display |
| **Reporting Module** | Outbound | Consumption data for compliance reports |

---

## 8. Acceptance Criteria

1. Water sources can be added, edited, and marked as legacy.
2. Meter readings calculate consumption correctly using delta method.
3. Water Balance updates automatically when new weather data arrives.
4. Monthly and yearly aggregations display accurate totals.
5. Legacy sources appear in historical queries but not in new entry forms.
6. Irrigation tasks integrate with schedule and labor tracking.

---

**Document End**
