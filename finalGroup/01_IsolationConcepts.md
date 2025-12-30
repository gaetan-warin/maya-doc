# Data Isolation & Identity Concepts
## Maya — Multi-Tenancy & Organizational Hierarchy

| Document Details | |
| :--- | :--- |
| **Project** | Maya Core 2.0 |
| **Module** | Identity & Access Management |
| **Scope** | Data Isolation Strategy |

---

## 1. Introduction

Maya addresses a complex requirement: supporting individual operational sites (e.g., a single golf course) while also managing large enterprise clusters (e.g., a franchise with 50 courses) and distributor networks.

To achieve this, the system uses a **Multi-Layered Isolation Strategy**. This ensures that data is strictly segregated where needed (Site Level) but aggregatable where authorized (Tenant/SuperTenant Level).

---

## 2. Hierarchy Levels (The "Big 3")

The three core entities defining data boundaries are **Group**, **Tenant**, and **Super Tenant**.

### 2.1 The Group (Operational Unit)
*   **What is it?** The atomic unit of operation. It represents a physical "Site" or "Facility" where work happens.
*   **Isolation Level:** **Strict.** Data like Tasks, Inventory, and Machinery are scoped to a Group.
*   **Data Partitioning:** Most operational tables (like `tasks`) use `idgroup` as the shard check.
*   **Analogy:** A single McDonalds restaurant.

### 2.2 The Tenant (Commercial Entity)
*   **What is it?** The paying customer / billable entity.
*   **Relationship:** One Tenant owns One or Many Groups.
*   **Role:** Configuration container. Settings, SSO configuration, and subscription packages live here.
*   **Analogy:** The Franchise Owner (who might own 5 restaurants).

### 2.3 The Super Tenant (Aggregator)
*   **What is it?** A virtual wrapper that aggregates multiple Tenants.
*   **Use Case:** Distributors, Resellers, or Holding Companies that need read-access or simplified management across completely separate Tenants.
*   **Mechanism:** It does not own data directly. It owns **Links** to Tenants.
*   **Analogy:** The Regional Manager or Distributor.

---

## 3. Isolation Logic

### How we prevent data leaks

| Layer | Partition Key | Responsibility |
| :--- | :--- | :--- |
| **API Layer** | `Bearer Token` | Identifies the User and their default context. |
| **App Logic** | `idgroup` | Every SQL query for operational data **MUST** include `WHERE idgroup = ?`. |
| **Tenant Logic** | `idtenant` | Configuration queries check `WHERE idtenant = ?`. |

### The "Default Group" Concept
Every Tenant has a `iddefault_group`. When a user logs in, they are typically placed into the context of this default group unless they explicitly switch.

---

## 4. Why This Structure?

### Scenario A: Single Course
*   1 Tenant = "Green Valley Golf"
*   1 Group = "Green Valley Golf Course"
*   **Result:** Simple 1:1 mapping.

### Scenario B: Multi-Course Resort
*   1 Tenant = "Grand Resort & Spa"
*   Group 1 = "North Course"
*   Group 2 = "South Course"
*   Group 3 = "Gardens"
*   **Result:** The resort manager handles billing (Tenant), but the North Course head greenkeeper only sees North Course tasks (Group).

### Scenario C: Distributor
*   Super Tenant = "Golf Supplies UK"
*   Tenant A = "Course 1"
*   Tenant B = "Course 2"
*   **Result:** The distributor can support multiple independent clients without those clients seeing each other's data.
