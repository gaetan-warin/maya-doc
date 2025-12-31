# Nutritional Inputs (Spraying)
## Functional Requirements Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-FRS-SPRAY-002 |
| **Version** | 2.0 (Premium) |
| **Last Updated** | December 31, 2025 |
| **Status** | Approved |
| **Owner** | Product Team |
| **Classification** | Internal - Functional |

---

## Executive Summary
The **Nutritional Inputs** module (commonly referred to as "Spraying") is the agronomic heart of the Maya platform. It empowers Turf Managers (Golf & Sports) to precisely control the application of fertilizers, biostimulants, and plant protection products.

Unlike simple inventory loggers, Maya's Spraying module is an **intelligent agronomic calculator**. It acts as a bridge between **Inventory Management** (stock deduction), **Fleet Operations** (machinery calibration), and **Agronomic Reporting** (NPK nutrient tracking). It supports a continuum of complexity, from simple "Log what I used" standard modes to "Precision Agronomy" modes that validate nozzle flow rates against tractor speeds.

**Key Value Propositions:**
*   **Precision Compliance:** Ensures legal compliance by tracking exact chemical inputs per hectare.
*   **Cost Control:** Real-time visualization of application costs prevents budget overruns.
*   **Agronomic Safety:** Built-in "Mulder's Rule" validation prevents dangerous chemical tank mixes.
*   **Operational Efficiency:** Auto-calculates tank mixes, water requirements, and staff workload.

---

## 1. Scope & Objectives

### 1.1 In Scope
*   **Task Management:** Creation, scheduling, and execution of spraying tasks.
*   **Tank Mix Logic:** Multi-product selection with antagonism checks.
*   **Inventory Integration:** Automatic stock deduction logic (FIFO).
*   **Machinery Calibration:** Nozzle selection, pressure, and speed validation.
*    **Reporting:** Elemental decomposition (NPK/Active Substance) and Cost Analysis.

### 1.2 Target Audience
*   **Head Greenkeeper / Grounds Manager:** Plans the nutrient strategy and approves budgets.
*   **Spray Technician:** Executes the task using specific machinery parameters (speed, pressure).
*   **General Manager:** Reviews cost and compliance reports.

---

## 2. Operational Modes
To cater to facilities of varying technical maturity, the module operates in three distinct modes, configurable at the tenant level.

### 2.1 Standard Mode (The "Logbook")
**Objective:** Simple, compliant record-keeping.
**User Journey:** "I just want to record that I put 20 bags of fertilizer on the Greens today."
*   **Inputs:** Date, Location, Product, Total Quantity.
*   **System Action:** Deducts stock and logs cost. No complex machinery math is required.

### 2.2 Advanced Spraying Mode (The "Operator")
**Objective:** Operational guidance for the spray technician.
**User Journey:** "I need to know how many tanks of water to mix and what speed to drive."
*   **Additional Inputs:**
    *   **Application Rate:** (e.g., 400 Liters/Hectare).
    *   **Tank Size:** Capacity of the sprayer (e.g., 600L).
*   **System Outputs:**
    *   **Total Dissolution Volume:** Calculates total water needed based on surface area.
    *   **Number of Tanks:** Tells the operator "You need 3.5 tanks to cover this area."
    *   **Recommended Flow:** Guides machine setup.

### 2.3 Precision Management Mode (The "Agronomist")
**Objective:** High-fidelity validation of application accuracy.
**User Journey:** "I want to ensure my nozzle selection matches my target rate and pressure."
*   **Additional Inputs:** Nozzle Color Code, Spacing (cm), Pressure (Bar).
*   **System Action:** Cross-references the **Nozzle Flow Rate** against the **Machine Speed** and **Target Rate**. If the physics don't align (e.g., driving too fast for the nozzle's output), the system flags the inefficiency.

---

## 3. Core Functional Workflows

### 3.1 Tank Mix Creation (The "Recipe")
When a user creates a spraying task, they are essentially building a chemical recipe.
1.  **Location Selection:** User selects a `Site` (Golf Course) and specific `Area` (e.g., "Greens 1-18").
    *   *Smart Area Logic:* If "Greens" are selected, the system sums the exact surface area of all greens to specific Hectare accuracy.
2.  **Product Selection:** User searches the catalog for products.
3.  **Mulder's Rule Validation (Safety Layer):**
    *   As products are added, the system analyzes their chemical composition.
    *   *Scenario:* User mixes a Calcium-rich product with a Phosphate-rich product.
    *   *Trigger:* System detects "Ca + P" antagonism (risk of precipitation/clogging).
    *   *Result:* Visible warning alert: *"Caution: Mixing Calcium and Phosphorus may cause precipitation."*
4.  **Quantity Calculation:**
    *   User inputs **Rate per Ha** (e.g., 20kg/Ha).
    *   System Auto-Calculates **Total Quantity** (e.g., 20kg * 1.5Ha = 30kg).

### 3.2 Execution & Stock Deduction
The lifecycle of a task determines its impact on Inventory.
*   **Planned (To Do):** reserves no stock. Allows for future planning without locking resources (Soft Allocation).
*   **In Progress / Done:** Hard Allocation. The moment a task is "Started", the stock is decremented from the specific inventory batch (FIFO logic).

---

## 4. Reporting & Analytics (The "Truth")

### 4.1 Elemental Decomposition (NPK)
Commercial products are rarely pure. A "20-10-10" fertilizer contains 20% Nitrogen, 10% Phosphorus, 10% Potassium.
*   **The Logic:** Maya decomposes every application into its elemental parts.
*   **The Value:** The user sees a report showing "Total Nitrogen Applied," not just "Total Bags Used." This is crucial for agronomic health monitoring (e.g., ensuring turf isn't "burned" by excess Nitrogen).

### 4.2 Cost Analysis
Every application carries a financial cost.
*   **Logic:** `Quantity Used` * `Unit Price` (weighted average cost from inventory).
*   **Granularity:** Costs can be analyzed per **Hole** (e.g., "Hole 14 costs 20% more to maintain") or per **Surface Type** ("Greens account for 60% of our chemical budget").

---

## 5. System Constraints & Rules

1.  **Unit Agnosticism:** The system is fully bi-lingual in units. It acts as a native Metric (Ha, L, Km/h) or Imperial (Acres, Gal, MPH) system based on tenant settings, handling all conversions invisibly.
2.  **Mandatory Surfaces:** You cannot spray "nothing." A valid surface area (from the Map module) is required for any rate-based calculation.
3.  **Date Integrity:** Operations cannot be logged in the future (except as "Planned"). Completed logs must have a timestamp ≤ Current Time.

---

**Document End**
