# Schedule Management
## Functional Requirements Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-FRS-SCHED-003 |
| **Version** | 3.1 (Deep Analysis & Business Rules) |
| **Last Updated** | December 31, 2025 |
| **Status** | Approved |
| **Owner** | Product Team |
| **Classification** | Internal - Functional |

---

## Executive Summary
The **Schedule Module** is the core operational engine of Maya. It manages the allocation of Staff and Machinery to specific agronomic tasks.

**Strategic Shift:** usage of the **Unified Task Model (Model 3)** to separate "Task Definition" from "Resourcing".
*   **Legacy Model:** "One Task = One Row". Assigning 3 people meant creating 3 database rows.
*   **Target Model:** "One Task = One Entity". A single Task ID holds multiple assignments.

---

## 1. Core Business Rules
These rules are hard constraints that the system must enforce to ensure data integrity across Inventory, Payroll, and Compliance.

### **BR-01: Mandatory Labor Tracking**
*   **Rule:** A Task cannot exist in a `Running` or `Completed` state without at least **one** assigned Staff member.
*   **Reason:** Costs must always be attributable to a human resource. An "Orphan Task" breaks the budget report.

### **BR-02: Action Definition**
*   **Rule:** Every Task must reference a valid `ActionID`. "Unspecified work" is not permitted.
*   **Constraint:** If an Action (e.g., "Mowing") is archived, existing history remains, but new tasks cannot select it.

### **BR-03: Inventory Deduction Trigger**
*   **Rule:** Stock is deducted **ONLY** when a Task transitions to `Running` (or `Completed`).
*   **Impact:** A "Planned" status acts as a *Soft Reservation* (viewable in "Stock Needs" report), but does not decrement the `Quantity in Hand`.

### **BR-04: Inventory Rollback Integrity**
*   **Rule:** If a `Running` or `Completed` task is Deleted or Reset to `Planned`, the system **MUST** refund the inventory.
*   **Mechanism:** A "Credit Transaction" is written to the `inventory_ledger`.

### **BR-05: Compliance Traceability (Spraying)**
*   **Rule:** Any task with `Type=SPRAYING` requires mandatory calibration data:
    *   Nozzle Type
    *   Application Speed
    *   Pressure
*   **Legal:** This data is immutable once the task is `Verified` (Locked).

### **BR-06: Legacy Dating Logic**
*   **Rule:** For legacy compatibility, a Task's "Planned Date" is initially determined by its creation timestamp if no specific start time is provided.

---

## 2. Functional Workflows & Concepts

### 2.1 The "Unified Task" Entity
A generic container for operational activity.
*   **Types:**
    *   `GENERAL`: Standard labour (e.g., Raking Bunkers).
    *   `SPRAYING`: Chemical application (Links to `task_ext_nutritional`).
    *   `LEAVE`: Non-productive time (Holidays/Sickness).
*   **Status Lifecycle:**
    1.  `DRAFT`: Created but not saved.
    2.  `PLANNED`: Saved in calendar.
    3.  `DISPATCHED`: Sent to mobile app.
    4.  `IN_PROGRESS`: Timer running.
    5.  `COMPLETED`: Finished.
    6.  `VERIFIED`: Audited by Manager (Locked).
    7.  `CANCELLED`: removed logically.

### 2.2 The "Merge Key" (Grouping Logic)
**Concept:** A mechanism to allow users to act on a "Batch" of tasks (e.g., "Mow All 18 Greens") as a single unit.
*   **Legacy Behavior:** Keys were string-based `(Action + Date + Site Group)`. Editing the date "broke" the key.
*   **Target Logic:** Grouping is relational.
    *   **Parent/Child:** A "Batch ID" links the 18 tasks.
    *   **Behavior:** Dragging the "Batch Card" updates all 18 linked `task_id`s in a single transaction.

---

## 3. Edge Cases & Known Pitfalls

### 3.1 Row Duplication (Legacy Only)
*   **Risk:** In the old system, assigning 3 staff created 3 completely independent rows.
*   **Impact:** If Manager A changed the "Time" on Row 1, Rows 2 and 3 might remain at the old time, creating "Split Brain" tasks.
*   **Mitigation:** The new UI forces Group Updates via the `Batch ID`, causing all linked assignments to stay synchronized.

### 3.2 "Ghost" Inventory Records
*   **Scenario:** A user edits a chemical task mid-operation (e.g., changes product from 'Iron' to 'Nitrogen').
*   **Risk:** The 'Iron' deduction might be orphaned if the system doesn't strictly effectively rollback the first transaction before applying the second.
*   **Solution:** Strict transactional boundaries (ACID) on all Edit operations.

---

**Document End**
