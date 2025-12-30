# Functional Specification Document (FSD)
## Maya — Task Management & Scheduling Module

| Document Details | |
| :--- | :--- |
| **Project** | Maya Core 2.0 |
| **Module** | Scheduling & Task Management |
| **Version** | 1.0 |
| **Status** | Authoritative Reference |
| **Target Audience** | Product Owners, Business Analysts, QA Engineers |

---

## 1. Introduction

### 1.1 Purpose
This document defines the **functional behavior** of the Maya Scheduling module. It serves as the contract between the Business and Development teams, describing *what* the system does, regardless of *how* it is implemented.

This specification is designed to:
1.  Provide a clear understanding of user capabilities.
2.  Define business rules that must be preserved during any refactoring.
3.  Serve as a baseline for QA acceptance criteria.

### 1.2 Scope

**In Scope:**
*   **Daily Scheduling (Day Plan):** Creation, modification, and deletion of scheduled tasks.
*   **Resource Allocation:** Assignment of Staff and Machinery to tasks.
*   **Nutritional Inputs (Spraying):** Recording of spraying/fertilization activities.
*   **Inventory Consumption:** Linking products to tasks and automated stock deduction.
*   **Time & Labor Management (TOIL):** Recording of overtime and time-off-in-lieu.

**Out of Scope:**
*   Fleet Management (Machines, Maintenance) — separate module.
*   Weather & Environmental Monitoring — separate module.
*   Financial Reporting & Invoicing — separate module.

---

## 2. Definitions & Glossary

| Term | Definition |
| :--- | :--- |
| **Task** | A unit of scheduled work. In the UI, this is a single item on the Day Plan. |
| **Task Cluster** | The internal grouping of data rows that represent a single user-facing Task. |
| **Action** | The type of work to be performed (e.g., "Mowing", "Aeration", "Topdressing"). |
| **Tenant** | The customer organization using Maya. |
| **Group** | A subdivision of a Tenant, representing an operational site (e.g., "North Course"). |
| **Site** | A top-level physical location (e.g., a Golf Course, a Stadium). |
| **Hole / Sub-site** | A specific area within a Site (e.g., "Hole 1", "Training Pitch"). |
| **Area (Tag)** | A granular work zone attribute applied to tasks (e.g., "Greens", "Fairways"). |
| **Spraying** | A specialized task for nutritional inputs with compliance/calibration data. |
| **TOIL (Time Off In Lieu)** | Accrued time off granted for overtime work. |

---

## 3. User Personas & Actors

| Actor | Role | Primary Goals |
| :--- | :--- | :--- |
| **Course Manager** | Supervises all operations | Create daily plans, assign resources, track completion. |
| **Head Greenkeeper** | Leads the grounds team | Execute tasks, update status, record issues. |
| **Greenkeeper / Staff** | Performs physical work | View assigned tasks, mark as complete. |
| **System Administrator** | Configures the platform | Manage Sites, Areas, Actions, Staff, Machines. |

---

## 4. Functional Requirements

### 4.1 Task Management (Scheduling)

| ID | Requirement |
| :--- | :--- |
| **FR-TM-01** | The system shall allow users to create a Task for a specific date. |
| **FR-TM-02** | A Task must specify an **Action** (what work is performed). |
| **FR-TM-03** | A Task must specify at least one **Location** (Site, and optionally Hole/Area). |
| **FR-TM-04** | A Task must have at least one **Staff** member assigned. |
| **FR-TM-05** | A Task may optionally have **Machinery** assigned. |
| **FR-TM-06** | A Task shall have a **Planned Duration** (estimate in minutes). |
| **FR-TM-07** | A Task may have **Planned Start/End Times**. |
| **FR-TM-08** | A Task may be flagged as **AM**, **PM**, or **Overtime**. |
| **FR-TM-09** | The system shall support **Multi-Site Tasks**, where a single logical job applies to multiple Sites simultaneously. |
| **FR-TM-10** | Users shall be able to **Edit** an existing Task (change Action, Location, Staff, Time). |
| **FR-TM-11** | Users shall be able to **Delete** a Task. |

### 4.2 Task Lifecycle & Status

| ID | Requirement |
| :--- | :--- |
| **FR-TL-01** | A Task shall have a **Status** (Planned, Running, Completed). |
| **FR-TL-02** | Users shall be able to toggle a Task to **Running** to indicate work has started. |
| **FR-TL-03** | When a Task is set to Running, the system shall record the **Actual Start Time**. |
| **FR-TL-04** | Users shall be able to toggle a Task to **Completed** to indicate work is finished. |
| **FR-TL-05** | When a Task is Completed, the system shall record the **Actual End Time**. |

### 4.3 Inventory & Product Consumption

| ID | Requirement |
| :--- | :--- |
| **FR-IP-01** | Users shall be able to link **Products** (consumables) to a Task. |
| **FR-IP-02** | When a Task transitions to **Running**, the system shall **deduct** the linked product quantities from the Inventory Ledger. |
| **FR-IP-03** | If a Task is reverted from Running to Planned, the system shall **refund** the inventory deduction. |
| **FR-IP-04** | The Inventory Ledger shall maintain an audit trail of all stock movements linked to Tasks. |

### 4.4 Nutritional Inputs (Spraying)

| ID | Requirement |
| :--- | :--- |
| **FR-SP-01** | The system shall support a **Spraying** task type with additional compliance fields. |
| **FR-SP-02** | Spraying records shall capture **Calibration Data** (Nozzle type, Speed, RPM, Pressure). |
| **FR-SP-03** | Spraying records shall capture **Product Application Data** (Quantity, Dosage per area). |
| **FR-SP-04** | Spraying records shall be visible alongside regular Tasks in the Calendar view. |

### 4.5 Time & Labor (TOIL / Overtime)

| ID | Requirement |
| :--- | :--- |
| **FR-WM-01** | Tasks flagged as **Overtime** shall contribute to TOIL accrual for assigned staff. |
| **FR-WM-02** | TOIL records shall be linked to the originating Task. |
| **FR-WM-03** | Users shall be able to view TOIL balances per staff member. |

---

## 5. Business Rules

These rules define the constraints that must be enforced by the system.

| ID | Rule | Rationale |
| :--- | :--- | :--- |
| **BR-01** | A Task cannot exist without at least one assigned Staff member. | Work must be attributed for labor tracking. |
| **BR-02** | A Task cannot exist without a specified Action. | The "what" is fundamental to scheduling. |
| **BR-03** | Inventory deduction occurs **only** when a Task is set to Running. | Prevents premature stock consumption. |
| **BR-04** | Deleting a Running/Completed Task shall reverse its inventory impact. | Maintains ledger integrity. |
| **BR-05** | Spraying records require compliance fields for regulatory traceability. | Health & Safety / Legal requirement. |
| **BR-06** | A Task's planned date is determined by its **Created On** date in the current system. | Legacy behavior to be aware of. |

---

## 6. Use Cases

### UC-01: Create a Daily Schedule

**Actor:** Course Manager
**Goal:** Plan work for the team for a specific day.
**Preconditions:** Sites, Staff, Actions, and Machines are configured.

**Main Flow:**
1.  User navigates to the Schedule page and selects a date.
2.  User clicks "Add Task".
3.  User selects an **Action** (e.g., "Mowing").
4.  User selects one or more **Locations** (Sites/Holes).
5.  User selects one or more **Areas/Tags** (e.g., "Greens").
6.  User assigns **Staff** members.
7.  User optionally assigns **Machinery**.
8.  User enters **Estimated Duration**.
9.  User saves the Task.
10. System persists the Task and displays it on the Day Plan.

---

### UC-02: Execute a Task

**Actor:** Head Greenkeeper
**Goal:** Record that work has started and consume materials.
**Preconditions:** A Task exists for today with linked Products.

**Main Flow:**
1.  User views the Day Plan.
2.  User identifies the target Task.
3.  User toggles the Task status to **Running**.
4.  **System Action:** Records the start timestamp.
5.  **System Action:** Deducts linked product quantities from Inventory.
6.  User performs the work.
7.  User toggles the Task status to **Completed**.
8.  **System Action:** Records the end timestamp.

**Exception Flow (Mistake):**
*   If the user accidentally set a Task to Running, they toggle it back.
*   **System Action:** Refunds the inventory deduction.

---

### UC-03: Record a Nutritional Input (Spraying)

**Actor:** Head Greenkeeper
**Goal:** Record a spraying application with full compliance data.
**Preconditions:** Products and Calibration settings are configured.

**Main Flow:**
1.  User navigates to the Spraying page.
2.  User creates a new Spraying record.
3.  User selects **Location** (Site/Holes/Areas).
4.  User selects **Product(s)** and enters **Quantities**.
5.  User enters **Calibration Data** (Nozzle, Speed, Pressure, etc.).
6.  User enters **Weather Conditions** (optional).
7.  User saves the record.
8.  System persists the Spraying record and deducts inventory.

---

## 7. Data Model (Conceptual)

This section provides a high-level view of the domain entities. For technical implementation details, see `02_TechnicalAudit_ASIS.md`.

```
┌─────────────┐       ┌───────────┐       ┌───────────┐
│   TENANT    │───────│   GROUP   │───────│   SITE    │
│ (Customer)  │ 1   * │ (OpUnit)  │ 1   * │ (Course)  │
└─────────────┘       └───────────┘       └───────────┘
                                                 │
                                                 │ 1
                                                 ▼ *
                                          ┌───────────┐
                                          │   HOLE    │
                                          │(Sub-site) │
                                          └───────────┘
┌───────────┐
│   TASK    │───────────────────────────────────────────────┐
│ (Job)     │                                               │
└───────────┘                                               │
      │                                                     │
      │ *                                                   │ *
      ▼ 1                                                   ▼ 1
┌───────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐
│  ACTION   │    │   STAFF   │    │  MACHINE  │    │  PRODUCT  │
│ (What)    │    │ (Who)     │    │ (With)    │    │(Consumable)│
└───────────┘    └───────────┘    └───────────┘    └───────────┘
```

---

## 8. Appendix: Calendar View Requirements

The Schedule Calendar displays a merged view of **Tasks** and **Spraying** records.

| Requirement | Description |
| :--- | :--- |
| **CV-01** | Calendar shall display items grouped by Date. |
| **CV-02** | Each item shall show the Action name and assigned Staff count. |
| **CV-03** | Items shall be visually distinguished by status (Planned, Running, Completed). |
| **CV-04** | Users shall be able to filter by Site, Action, or Staff. |
