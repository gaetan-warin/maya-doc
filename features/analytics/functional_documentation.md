# Reports & Analytics
## Functional Requirements Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-FRS-ANLY-001 |
| **Version** | 1.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Product Team |
| **Classification** | Internal |

---

## Executive Summary

The **Reports & Analytics** module provides comprehensive data visualization, analysis, and reporting capabilities across all operational areas. It encompasses four integrated tools: **Soil Analysis** (laboratory results), **Graph Builder** (custom visualizations), **Interactive Report** (multi-dimensional analysis), and **History** (activity logs).

This document defines the functional requirements for data exploration, custom chart creation, cross-functional reporting, and historical data export. It integrates with Weather, Agronomy, Fleet, Water, and Task modules to provide unified operational insights.

**Key Business Outcomes:**
- **Data-Driven Decisions**: Visualize trends to optimize operations.
- **Performance Tracking**: Compare planned vs. actual metrics across departments.
- **Compliance Reporting**: Generate auditable records of all activities.
- **Cost Analysis**: Track spending by category, cost center, and time period.

---

## 1. Scope & Objectives

### 1.1 In Scope
- Soil analysis result management and export
- Custom graph creation with multiple metrics
- Interactive multi-tab reporting
- Historical activity and cost tracking
- CSV/Excel/PDF export capabilities

### 1.2 Out of Scope
- Scheduled report generation (future enhancement)
- External business intelligence tool integration
- Real-time dashboard streaming

### 1.3 Target Users
| Role | Primary Use Case |
|:---|:---|
| **Superintendent** | Operational performance review |
| **Agronomist** | Soil and nutritional analysis |
| **Finance Manager** | Cost tracking and budget analysis |
| **Operations Manager** | Staff and fleet utilization |

---

## 2. Soil Analysis

### 2.1 Overview

Soil Analysis provides access to professional laboratory results uploaded by the Maya team. Users can review, filter, and export soil health data.

### 2.2 Filters

| Filter | Description |
|:---|:---|
| **Substance** | Chemical or nutrient (pH, N, P, K, etc.) |
| **Site** | Location filter |
| **Area** | Zone within site |
| **Depth** | Soil sampling depth |

### 2.3 Data Table Columns

| Column | Description |
|:---|:---|
| Date | Analysis date |
| Site | Location |
| Name | Analysis name/description |
| Type | Analysis type |
| Settings | Actions menu |

### 2.4 Export Options

| Action | Format | Description |
|:---|:---|:---|
| **Excel** | .xlsx | Full data export |
| **PDF Result** | .pdf | Formatted analysis report |

> [!NOTE]
> Professional soil analysis results are uploaded by the Maya team upon receipt from laboratories.

---

## 3. Graph Builder

### 3.1 Overview

Graph Builder enables custom data visualization with flexible axis configuration, metric selection, and template saving.

### 3.2 Tabs

| Tab | Purpose |
|:---|:---|
| **Graph Builder** | Create and customize visualizations |
| **Graph Statistics** | View statistical summaries |

### 3.3 Metric Categories

| Category | Description | Example Metrics |
|:---|:---|:---|
| **Environment Metrics** | Weather and soil data | Temperature, Humidity, Soil Moisture |
| **Performance Metrics** | Operational KPIs | ET, DLI, Growth Potential |
| **Nutritional Inputs** | Application data | N/P/K applied, product usage |
| **Tasks** | Activity data | Completed tasks, labor hours |

### 3.4 Graph Configuration

| Setting | Options |
|:---|:---|
| **Left Y-Axis** | Select metric for left axis |
| **Right Y-Axis** | Select metric for right axis |
| **Device** | Select specific sensor/source |
| **Frequency** | Hourly, Daily, Raw Data, Daily Cumulative |

### 3.5 Template Management

| Action | Description |
|:---|:---|
| **Save Setup** | Store current configuration |
| **Load Template** | Apply saved configuration |

### 3.6 Time Controls

- Custom date range selection
- Quick presets: Last 30 days, Year to date
- Flexible start/end date pickers

### 3.7 Graph Statistics

High-level statistical cards showing:
- Mean, Min, Max values
- Trend indicators
- Export to Excel option

---

## 4. Interactive Report

### 4.1 Overview

Interactive Report provides multi-dimensional analysis across six operational areas with flexible filtering and visualization.

### 4.2 Global Controls

| Control | Description |
|:---|:---|
| **Date Range** | Start and end date selection |
| **Quick Selectors** | Year to date, Last quarter, This month |
| **Generate** | Refresh report data |
| **Print** | Print-friendly output |
| **Export Data** | Download as file |

### 4.3 Report Tabs

#### 4.3.1 Human Resource Tab

| Data | Description |
|:---|:---|
| Staff Hours | Planned vs. Actual comparison |
| Visualization | Pie/Bar charts |
| Filters | Site, Area |

#### 4.3.2 Activity Tab

| Data | Description |
|:---|:---|
| Task Summary | Activity durations |
| Comparison | Actual vs. Default duration |
| Categories | Activity types |

#### 4.3.3 Fleet Tab

| Data | Description |
|:---|:---|
| Machine Usage | Cost and activity counts |
| Filters | Machine Category, Machine Tags |
| Metrics | Total cost, usage hours |

#### 4.3.4 Water Tab

| Data | Description |
|:---|:---|
| Input Flow | Total water input by date |
| Filters | Site selection |
| Visualization | Flow over time |

#### 4.3.5 Nutritional Inputs Tab

| Data | Description |
|:---|:---|
| Product Usage | Cost breakdown by product |
| Totals | Total cost in currency |
| Details | Product names, quantities |

#### 4.3.6 Assets Tab

| Data | Description |
|:---|:---|
| Inventory Movement | In/Out/Change values |
| Stock Types | Product, Machinery part |
| Filters | Stock type selection |

---

## 5. History

### 5.1 Overview

History provides a comprehensive log of all activities, fertilizations, and operational events with flexible filtering and export.

### 5.2 Filters

| Filter | Description |
|:---|:---|
| **Sites** | Multi-select location filter |
| **Type** | Activity/event type |
| **Areas** | Zone filter |
| **Activity** | Specific activity filter |

### 5.3 View Modes

| Mode | Description |
|:---|:---|
| **View Report** | Activity-centric view |
| **View Staff** | Staff-centric view |

### 5.4 Data Table Columns

| Column | Description |
|:---|:---|
| Date | Activity date |
| Type | Activity type |
| Site | Location |
| Zones | Affected areas |
| Staff | Assigned personnel |
| Activity | Activity description |
| Cost | Associated cost |
| Duration | Time spent |

### 5.5 Export Options

| Action | Format | Description |
|:---|:---|:---|
| **Export CSV** | .csv | Full data export |
| **Print** | Printable view | Print-friendly format |

---

## 6. Integration Points

| System | Integration Type | Data Flow |
|:---|:---|:---|
| **Weather Module** | Inbound | Environmental metrics |
| **Agronomy Module** | Inbound | Disease and growth data |
| **Water Module** | Inbound | Irrigation flow data |
| **Fleet Module** | Inbound | Machine usage |
| **Task Module** | Inbound | Activity and labor data |
| **Stock Module** | Inbound | Inventory movements |

---

## 7. Acceptance Criteria

1. Soil analysis results display with all configured filters.
2. Graph Builder creates visualizations with selected metrics and dual Y-axis.
3. Saved templates can be loaded and applied.
4. Interactive Report loads data for all six tabs when date range is selected.
5. History filters update table results dynamically.
6. All export functions generate valid files (CSV, Excel, PDF).
7. Print functionality renders clean, printable output.

---

**Document End**
