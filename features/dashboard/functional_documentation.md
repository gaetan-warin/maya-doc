# Dashboard & Task Management
## Functional Requirements Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-FRS-DASH-001 |
| **Version** | 3.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Product Team |
| **Classification** | Internal |

---

## Executive Summary

The **Dashboard & Task Management** module is Maya's operational command center. It enables turf management professionals to plan, assign, execute, and analyze all ground-level work across golf courses, football stadiums, and multi-sport venues.

This document defines the functional requirements for task scheduling, labor tracking, and operational reporting. It establishes the **Unified Task Model**—a standardized framework ensuring consistent behavior across all work types (mowing, fertilizing, spraying, repairs, etc.) regardless of venue type.

**Key Business Outcomes:**
- **Operational Efficiency**: Centralized planning reduces coordination overhead by 40%.
- **Labor Accountability**: Real-time mobile tracking ensures accurate timesheets.
- **Cost Visibility**: Automatic labor cost calculations enable budget compliance.

---

## 1. Scope & Objectives

### 1.1 In Scope
- Task creation, assignment, and scheduling
- Staff and machinery allocation
- Mobile-based time tracking (start/stop)
- Planned vs. Actual labor analysis
- Overtime management and reporting

### 1.2 Out of Scope
- Payroll integration (handled by external HR systems)
- Equipment maintenance scheduling (see Machinery Module)

### 1.3 Target Users
| Role | Primary Use Case |
|:---|:---|
| **Head Greenkeeper / Superintendent** | Daily planning, resource allocation, performance review |
| **Grounds Staff** | Task execution via mobile app |
| **Operations Manager** | Labor cost analysis, compliance reporting |

---

## 2. The Unified Task Model

### 2.1 Definition
All operational work in Maya follows a single, standardized structure:

> **A Task = Planned Work + Actual Work**, executed by assigned Staff using designated Machinery, across one or more Locations.

This model applies uniformly to:
- Mowing, aeration, verticutting
- Fertilizing, spraying, topdressing
- Hand-watering, irrigation checks
- Line marking, bunker raking
- Repairs and renovations

### 2.2 Planned Duration Logic

The **Planned Duration** represents the expected work time **per staff member**, not the aggregate labor.

**Formula:**
```
Total Planned Labor Hours = Planned Duration × Number of Assigned Staff
```

**Example:**
| Scenario | Start | End | Duration | Staff | Total Labor |
|:---|:---|:---|:---|:---|:---|
| Aeration (Golf) | 08:00 | 11:00 | 3 hours | 2 | **6 labor hours** |
| Pitch Grooming (Football) | 07:00 | 09:00 | 2 hours | 3 | **6 labor hours** |

### 2.3 Multi-Location Distribution

When a single task covers multiple locations (e.g., "Mow Greens 1–18" or "Groom Pitches A + B"), the system **distributes duration evenly** across all selected locations.

**Rationale:**
- Staff log a single work session on mobile; there is no facility to log time per sub-location.
- Even distribution enables consistent cost attribution to individual assets (holes, pitches, zones).
- Users requiring location-specific durations must create separate tasks.

---

## 3. Labor Tracking & Timekeeping

### 3.1 Mobile Execution Flow

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   ASSIGNED  │───▶│   STARTED   │───▶│ IN PROGRESS │───▶│  COMPLETED  │
│   (todo)    │    │(task_start) │    │  (running)  │    │  (task_end) │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

Each staff member independently clocks in/out via the mobile application. This produces **actual hours per person**, which override planning assumptions.

### 3.2 Status Definitions

| Status | Description | Data State |
|:---|:---|:---|
| `todo` | Task scheduled but not started | `running=0`, `task_start=null` |
| `in_progress` | At least one staff member has begun work | `running=0`, `task_start=value` |
| `completed` | All assigned staff have finished | `running=1`, `task_start=value`, `task_end=value` |
| `not_completed` | Task stopped without full completion | `running=0`, `task_start=value`, `task_end=value` |

### 3.3 Overtime Calculation

The system supports two overtime models:

| Model | Trigger | Calculation |
|:---|:---|:---|
| **Flagged Overtime** | Task explicitly marked as "OT" | 100% of task duration = overtime |
| **Schedule Overtime** | Work extends beyond staff's daily schedule | Only excess hours = overtime |

**Example (Schedule Overtime):**
Staff scheduled to finish at 16:00. Task runs 14:00–18:00.
- Regular hours: 14:00–16:00 (2 hours)
- Overtime hours: 16:00–18:00 (2 hours)

---

## 4. Machinery & Staff Assignment

### 4.1 Assignment Rules

| Scenario | Requirement |
|:---|:---|
| **Spraying Operations** | Mandatory 1:1 assignment (Operator → Sprayer) for legal traceability |
| **Shared Transport** | Multiple staff may share utility vehicles |
| **Solo Operations** | Single staff member operates single machine |

### 4.2 Compliance Considerations

Machine-to-operator linkage is essential for:
- Chemical application audit trails
- Health & Safety incident investigation
- Insurance liability documentation

---

## 5. Stock Consumption

Tasks may consume inventory items. The consumption type depends on the task category:

| Task Type | Consumes | Examples |
|:---|:---|:---|
| **Nutritional Input** | Products | Fertilizers, herbicides, fungicides, wetting agents |
| **Standard Task** | Non-Products | Fuel, sand, stakes, marking paint, repair materials |

---

## 6. User Interface Components

### 6.1 Day Plan View
- Gantt-style visualization of daily task assignments
- AM/PM slot buckets for quick scheduling
- Drag-and-drop task allocation

### 6.2 Status Indicators
- Real-time icons showing staff work status (Active / Completed / Pending)
- Color-coded priority flags

### 6.3 Quick Actions
- Add Task
- Duplicate Task
- Reassign Staff
- Mark Complete

---

## 7. Integration Points

| System | Integration Type | Data Flow |
|:---|:---|:---|
| **Mobile App** | Real-time sync | Time logs, GPS location, task updates |
| **Reporting Module** | Data consumer | Labor hours, cost summaries, efficiency metrics |
| **Stock Module** | Transactional | Consumption deduction upon task completion |
| **Weather Module** | Reference | Spray window recommendations |

---

## 8. Acceptance Criteria

1. Tasks created via web UI appear immediately on assigned staff mobile devices.
2. Mobile start/stop actions update task status in real-time on web dashboard.
3. Overtime is calculated correctly for both flagged and schedule-based scenarios.
4. Multi-site tasks distribute labor evenly across all selected locations.
5. Stock consumption is recorded upon task completion.

---

**Document End**
