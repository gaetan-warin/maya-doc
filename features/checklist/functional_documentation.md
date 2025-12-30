# MyChecklist
## Functional Requirements Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-FRS-CHECK-001 |
| **Version** | 1.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Product Team |
| **Classification** | Internal |

---

## Executive Summary

The **MyChecklist** module enables creation, assignment, and tracking of standardized inspection checklists for fleet machinery. It ensures consistent pre-operation checks, safety compliance, and maintains an audit trail of all completed inspections.

This document defines the functional requirements for checklist template creation, machine assignment, execution tracking, and historical reporting. It integrates with Fleet Management to ensure machinery receives appropriate inspections before use.

**Key Business Outcomes:**
- **Safety Compliance**: Ensures pre-op checks are completed before equipment use.
- **Consistency**: Standardized inspection procedures across all operators.
- **Audit Trail**: Complete history of all inspections for regulatory compliance.
- **Accountability**: Clear assignment of inspection responsibilities.

---

## 1. Scope & Objectives

### 1.1 In Scope
- Checklist template creation and management
- Assignment of checklists to specific machines
- Execution tracking (item completion status)
- Historical record of completed checklists
- Integration with machinery hour logging

### 1.2 Out of Scope
- Real-time alerting for missed checklists
- Automated scheduling (tied to calendar)
- Photo/signature capture (mobile app feature)

### 1.3 Target Users
| Role | Primary Use Case |
|:---|:---|
| **Fleet Manager** | Template creation, assignment configuration |
| **Workshop Supervisor** | Assignment of checklists to machines |
| **Equipment Operator** | Checklist execution before machine use |
| **Compliance Officer** | History review for audits |

---

## 2. Module Navigation

The MyChecklist module contains three primary tabs:

| Tab | Purpose |
|:---|:---|
| **My Checklists** | Create and manage checklist templates |
| **Checklist Manager** | Assign checklists to specific machines |
| **Checklists History** | View completed inspection records |

---

## 3. My Checklists (Template Management)

### 3.1 Checklist Template Structure

| Field | Description |
|:---|:---|
| Checklist Name | Descriptive title (e.g., "Daily Mower Pre-Op Check") |
| Actions | Individual inspection items |
| Machinery Hour Check | Boolean: Require hour reading on completion |
| Is Active | Template availability status |

### 3.2 Action Item Structure

Each checklist contains one or more action items:

| Field | Description |
|:---|:---|
| Action Description | What to inspect/verify |
| Order | Display sequence |
| Is Required | Must be completed to submit |

### 3.3 Template Actions

| Action | Description |
|:---|:---|
| **Create List** | Add new checklist template |
| **Edit** | Modify existing template |
| **Delete** | Remove template (if no active assignments) |
| **Duplicate** | Copy template for modification |

### 3.4 Example Checklist

**Checklist Name**: Daily Mower Pre-Op Check

| # | Action Item | Required |
|:---|:---|:---|
| 1 | Check oil level | Yes |
| 2 | Inspect blade condition | Yes |
| 3 | Verify tire pressure | Yes |
| 4 | Test safety switches | Yes |
| 5 | Check fuel level | No |
| 6 | Note any visible damage | No |

---

## 4. Checklist Manager (Assignment)

### 4.1 Assignment View

The Checklist Manager displays:
- List of all machinery with assigned checklists
- Search functionality by machine name
- Pagination for large fleets

### 4.2 Machine Assignment

| Field | Description |
|:---|:---|
| Machine | Target equipment |
| Assigned Checklists | One or more checklist templates |
| Frequency | Per use / Daily / Weekly (future) |

### 4.3 Assignment Actions

| Action | Description |
|:---|:---|
| **Assign Checklist** | Link template to machine |
| **Change Assignment** | Modify existing assignments |
| **Remove Assignment** | Unlink checklist from machine |

---

## 5. Checklists History

### 5.1 History View

Displays all completed checklist submissions:

| Column | Description |
|:---|:---|
| Machine Name | Equipment checked |
| Checklist Name | Template used |
| Completed By | Operator who submitted |
| Completed Date | Submission timestamp |
| Status | Complete / Partial |

### 5.2 Sorting Options

| Sort Field | Description |
|:---|:---|
| Date | Chronological order |
| Machine | Alphabetical by machine |
| Operator | By submitter name |

### 5.3 History Detail View

Clicking a history record shows:
- Individual item completion status
- Notes entered for each item
- Machinery hours at time of check
- Photos attached (if applicable)

---

## 6. Execution Flow

### 6.1 Operator Workflow

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ Select      │───▶│ Complete    │───▶│ Enter Hours │───▶│   Submit    │
│ Machine     │    │ All Items   │    │ (if reqd)   │    │             │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

### 6.2 Item Completion

| Status | Icon | Meaning |
|:---|:---|:---|
| Pending | ○ | Not yet checked |
| Complete | ✓ | Item verified OK |
| Failed | ✗ | Issue found (triggers note) |

### 6.3 Submission Rules

- All required items must be completed before submission
- Hours entry required if template specifies "Machinery Hour Check"
- Failed items require explanatory notes
- Submission creates history record

---

## 7. Integration with Fleet Management

### 7.1 Machine Detail Integration

From the Machinery Detail page:
- **Manage Checklist** button opens assignment modal
- View assigned checklists for this machine
- Access checklist history specific to this machine

### 7.2 Hour Logging Integration

When a checklist with "Machinery Hour Check" is submitted:
- Hours reading is captured
- Machine's total hours are updated automatically
- Creates hour log entry in maintenance history

---

## 8. Acceptance Criteria

1. New checklist templates can be created with multiple action items.
2. Templates can be assigned to one or more machines.
3. Operators can complete checklists and submit with required fields.
4. Completed checklists appear in history with all item details.
5. History records cannot be modified after submission.
6. Hour readings from checklists update machine total hours.
7. Search in Checklist Manager returns matching machines.

---

**Document End**
