# Task, Labour & Nutritional Input (Schedule/Team Management Refactoring)

## TO-BE Model (Golf & Football)

### 1. Introduction
Maya is used daily in golf courses, football stadiums and multisport venues to plan, execute and analyse operational work.

To remain credible, compliant, and scalable, we must define one single unified logic for:
- Tasks
- Nutritional inputs
- Staff
- Machines
- Sites, holes, pitches, and zones
- Planned vs actual hours
- Overtime
- Reporting and analytics
- Stock usage and legal compliance

This document describes the TO-BE behaviour, the future, corrected, consistent model, and finishes with the AS-IS inconsistencies currently being fixed.

---

### 2. Core Concepts: One Model for All Work

Across golf and football, all operational work falls into the same structure:

**A task = planned work + actual work**, carried out by specific staff, machines, and parameters, across one or multiple locations.

This includes:
- Mowing, fertilising, spraying, topdressing, bunker raking
- Hand-watering, line marking, grooming artificial pitches
- Aeration, verticutting, repairs, renovations
- Irrigation checks, inspections
- All nutritional/chemical/amendment applications (fertilisers, wetting agents, PGRs, herbicides, fungicides, etc.)

---

### 3. Planned Duration — What It Means

#### 3.1 One task = one planned duration
The planned duration expresses the **expected time per staff member**, not the total labour.

**Example (Golf):**
Aeration of 5 greens on Course A (8am → 11am = 3h), 2 staff.
- Planned labour = 3h × 2 = **6 labour hours**

**Example (Football):**
Morning pitch grooming (7am → 9am = 2h), 3 groundskeepers.
- Planned labour = 2h × 3 = **6 labour hours**

---

### 6. Multi-Site / Multi-Hole / Multi-Zone Tasks (Critical Rule)

**A staff member logs one work session, not multiple.**

When a task covers multiple locations, the real duration is always split evenly across each location.

**Example:**
- Topdressing Greens 1, 2, 3
- Staff: 1
- Mobile duration: 3 hours
- Per-green distribution: **1h per green**

---

### 8. Overtime — TO-BE Logic

#### 8.1 Overtime due to the task being flagged as OT
When a task is explicitly marked as overtime work.

#### 8.2 Overtime due to exceeding the staff member's daily schedule
When actual hours worked exceed the contracted hours for that day.

---

### 9. Nutritional Inputs (Fertilisation / Spraying) — TO-BE Normalisation

Nutritional inputs should be a **task with additional parameters**:
- Nozzle type
- Tractor speed
- RPM
- Pressure
- Tank size
- etc.

**Stock Logic:**
- Nutritional inputs consume stock items of type **"Product"**
- Tasks consume stock items of type **"Non-product"**

---

(Archived from Notion on 2025-12-22)
