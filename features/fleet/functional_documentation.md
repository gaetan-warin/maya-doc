# Fleet Management
## Functional Requirements Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-FRS-FLEET-001 |
| **Version** | 1.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Product Team |
| **Classification** | Internal |

---

## Executive Summary

The **Fleet Management** module provides comprehensive lifecycle management for all machinery and equipment assets across golf courses, stadiums, and sports venues. It enables operations teams to track usage hours, schedule preventive maintenance, manage incidents, monitor fuel consumption, and analyze total cost of ownership.

This document defines the functional requirements for machinery tracking, maintenance workflow management, and fleet analytics. It establishes a Kanban-based maintenance system with clear status progression and integrates with the Stock module for parts management.

**Key Business Outcomes:**
- **Reduced Downtime**: Proactive maintenance scheduling reduces unplanned equipment failures by 40%.
- **Cost Visibility**: Total cost of ownership tracking enables data-driven replacement decisions.
- **Compliance**: Complete maintenance history provides audit trail for insurance and safety compliance.
- **Efficiency**: QR code integration enables rapid field identification and status updates.

---

## 1. Scope & Objectives

### 1.1 In Scope
- Machinery registration and asset management
- Usage hours tracking and logging
- Maintenance scheduling and workflow (Kanban)
- Fuel consumption monitoring
- Parts and consumables tracking
- Machine transfer between tenants
- QR code generation for field identification
- Fleet cost analytics and reporting

### 1.2 Out of Scope
- GPS/Telematics real-time tracking
- Automated IoT sensor integration
- Procurement and purchasing workflows

### 1.3 Target Users
| Role | Primary Use Case |
|:---|:---|
| **Fleet Manager** | Overall fleet oversight, cost analysis, replacement planning |
| **Workshop Technician** | Maintenance execution, parts consumption, hour logging |
| **Grounds Staff** | Daily pre-op checks, hour logging, incident reporting |
| **Operations Manager** | Budget analysis, depreciation tracking |

---

## 2. Fleet Dashboard Overview

### 2.1 Status Metrics

The dashboard displays real-time fleet status aggregations:

| Metric | Description |
|:---|:---|
| **Total Machines** | Count of all registered assets |
| **In Use** | Machines currently assigned to active tasks |
| **In Service** | Machines available and operational |
| **Due Maintenance** | Machines requiring scheduled service |
| **Out of Service** | Machines currently non-operational |

### 2.2 Financial Metrics

| Metric | Description | Calculation |
|:---|:---|:---|
| **Monthly Cost** | Total maintenance + parts cost this month | Sum of completed maintenance costs |
| **Yearly Cost** | Total cost year-to-date | Sum of all costs since Jan 1 |
| **Average Fleet Age** | Mean age of all assets | Current year − average purchase year |

---

## 3. View Modes

### 3.1 Kanban View (Default)

A visual board displaying maintenance items organized by status:

| Column | Purpose | Card Contents |
|:---|:---|:---|
| **Urgent** | Critical items requiring immediate attention | Issue name, machine, hours, due date |
| **Minor** | Low-priority maintenance items | Issue name, machine, hours |
| **Due Maintenance** | Scheduled maintenance now due | Service type, machine, hours since last |
| **Upcoming** | Future scheduled maintenance | Service type, machine, scheduled date |

### 3.2 List View

A tabular view of all machinery with sortable columns:

| Column | Description |
|:---|:---|
| Machine Name | User-defined identifier |
| Brand | Manufacturer (Toro, John Deere, etc.) |
| Category | Equipment type (Tractor, Mower, Utility) |
| Purchase Year | Acquisition year |
| Total Hours | Lifetime usage hours |
| Status | Current operational status |
| Tags | User-defined classification tags |

---

## 4. Machinery Detail Page

### 4.1 Asset Information

| Field | Description |
|:---|:---|
| Brand / Model | Manufacturer and model designation |
| Frame Number | Serial/VIN number |
| Fleet Number | Internal asset number |
| Purchase Date | Acquisition date |
| Purchase Price | Original cost |
| Current Value | Depreciated value (if calculated) |
| Category | Equipment classification |
| Ownership | Owned / Leased / Rented |

### 4.2 Usage Tracking

| Metric | Description |
|:---|:---|
| Total Hours | Lifetime cumulative hours |
| Hours This Year | YTD usage |
| Last Entry | Date of most recent hour log |
| Average Daily | Calculated daily usage rate |

### 4.3 Maintenance Tabs

| Tab | Contents |
|:---|:---|
| **Urgent** | Critical issues requiring immediate attention |
| **Minor** | Low-priority items |
| **Due** | Maintenance items now due |
| **Upcoming** | Future scheduled maintenance |
| **History** | Completed maintenance records |
| **Missed** | Overdue maintenance not completed |

### 4.4 Machine Actions

| Action | Description |
|:---|:---|
| **Add Hours** | Log usage hours manually |
| **Add Fuel** | Record fuel consumption |
| **New Maintenance** | Create maintenance record |
| **Generate QR** | Create QR code for field scanning |
| **Manage Checklist** | Assign pre-op checklists |
| **Machine Files** | Upload manuals/documents |
| **Transfer Machine** | Move asset to different tenant |

---

## 5. Maintenance Workflow

### 5.1 Maintenance Types

| Type | Trigger | Example |
|:---|:---|:---|
| **Scheduled** | Hours-based or calendar-based | Oil change every 250 hours |
| **Recurring** | Automatic regeneration | Annual service |
| **Incident** | Reported issue | Hydraulic leak |
| **Inspection** | Regulatory requirement | Annual safety check |

### 5.2 Status Lifecycle

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  CREATED │───▶│   DUE    │───▶│IN PROGRESS│───▶│ COMPLETED│
└──────────┘    └──────────┘    └──────────┘    └──────────┘
                     │                               │
                     ▼                               ▼
               ┌──────────┐                    ┌──────────┐
               │  MISSED  │                    │ HISTORY  │
               └──────────┘                    └──────────┘
```

### 5.3 Maintenance Record Fields

| Field | Description |
|:---|:---|
| Issue / Service | Description of work required |
| Priority | Urgent / Minor / Scheduled |
| Assigned To | Responsible technician |
| Due Date | Target completion date |
| Due Hours | Hours threshold trigger |
| Parts Used | Linked stock items consumed |
| Labor Hours | Time spent on repair |
| Cost | Total maintenance cost |
| Notes | Free-text observations |

---

## 6. Fuel Management

### 6.1 Fuel Entry Fields

| Field | Description |
|:---|:---|
| Date | Fill date |
| Fuel Type | Diesel / Petrol / Electric |
| Quantity | Liters added |
| Cost | Fuel cost (optional) |
| Odometer/Hours | Reading at fill time |

### 6.2 Fuel Analytics

| Metric | Description |
|:---|:---|
| Average Consumption | Liters per hour |
| Monthly Usage | Total fuel this month |
| Cost per Hour | Calculated fuel expense rate |

---

## 7. Integration Points

| System | Integration Type | Data Flow |
|:---|:---|:---|
| **Stock Module** | Transactional | Parts consumption deduction |
| **Task Module** | Reference | Machine assignment to tasks |
| **MyChecklist** | Linked | Pre-op inspection assignments |
| **Reporting** | Consumer | Fleet cost reports |

---

## 8. Acceptance Criteria

1. New machinery can be registered with all required fields and appears in fleet list.
2. Hours logged via mobile or web update the machine's total hours immediately.
3. Maintenance items move through Kanban columns based on status changes.
4. Completed maintenance generates history record with cost summary.
5. QR code generation produces scannable code linking to machine detail.
6. Fuel entries update average consumption calculations.
7. Machine transfer updates tenant association and preserves history.

---

**Document End**
