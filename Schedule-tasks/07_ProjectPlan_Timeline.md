# Project Planning & Timeline
## Maya Scheduling Migration (Model 2 → Model 3)

| Document Details | |
| :--- | :--- |
| **Project** | Maya Core 2.0 |
| **Module** | Scheduling & Task Management |
| **Timeline** | **8 Weeks** (Risk-Adjusted) |
| **Team Size** | 3 Core Members |

---

## 1. Executive Summary

This project migrates the Maya Scheduling Module from a fragile, denormalized legacy architecture to a robust, normalized Enterprise Standard ("Model 3").

**Key Constraints:**
*   **High Data Volume:** Requires background migration strategies.
*   **Team Velocity:** Timeline includes buffer for upskilling and careful validation.
*   **Zero Downtime:** Business operations must continue uninterrupted.

**Target Delivery:** 8 Weeks (4 Sprints of 2 weeks).

---

## 2. Resource Plan

| Role | Count | Responsibilities | Requirements |
| :--- | :--- | :--- | :--- |
| **Tech Lead / Senior Backend** | 1 | Architecture ownership, Code Review, Complex Logic (Data Migration, Synchronization). | Laravel Expert, SQL Optimization, Legacy Refactoring experience. |
| **Frontend Developer** | 1 | Migrating UI to new V3 Endpoints, removing logic from JS. | Vue.js, State Management, API Integration. |
| **QA / Tester** | 1 | Regression testing, Data Parity Validation (`Legacy vs New`). | SQL for data checking, Manual UI testing. |

---

## 3. Work Breakdown Structure (WBS)

### Phase 1: Foundation & Safety Net (Weeks 1-2)
**Goal:** Prepare the system for dual-write without breaking existing features.

*   **Week 1: Setup & Migrations**
    *   [Backend] Create Model 3 Migrations (`tasks`, `assignments`, etc.).
    *   [Backend] Set up `TaskService` (V3 Service Layer).
    *   [Team] Deep Dive: Verify `merge_key` logic one last time.
*   **Week 2: Dual-Write Implementation**
    *   [Backend] Implement Dual-Write in `ScheduleController`.
        *   *Write to Legacy → Success → Write to New (Try/Catch Log).*
    *   [DevOps] Deploy Schema to Production.

**Milestone 1:** Production DB has new tables, and new data is populating both systems.

---

### Phase 2: The Core Refactor (Weeks 3-5)
**Goal:** Build the new logic and migrate historical data.

*   **Week 3: Heavy Backend Logic**
    *   [Backend] Implement V3 Read Endpoints (`GET /api/v3/tasks`).
    *   [Backend] Optimize Read Queries (Eager Loading, Indexes).
*   **Week 4: Data Migration Script**
    *   [Backend] Write Batch Migration Command (`chunk(1000)`).
    *   [QA] Test Migration on Staging Copy options.
*   **Week 5: Execution & Staging**
    *   [Backend] Start Background Migration in Production (Example: 50% capacity at night).
    *   [Frontend] Begin wiring V3 Endpoints behind Feature Flag.

**Milestone 2:** All Historical Data is in V3 format. Frontend works on Staging with V3.

---

### Phase 3: Switchover (Weeks 6-7)
**Goal:** Move users to the new system.

*   **Week 6: Frontend Finalization**
    *   [Frontend] complete V3 integration.
    *   [Frontend] Remove `merge_key` generation logic from client-side (use Backend ID).
*   **Week 7: The "Read Switch"**
    *   [DevOps] Enable `FEATURE_UNIFIED_TASK_READ = true` for 10% of users (Canary).
    *   [QA] Monitor Error Logs & Performance.
    *   [DevOps] Ramp up to 100% users.

**Milestone 3:** Users are reading from V3 tables. Legacy tables are "Write-Only" (deprecated).

---

### Phase 4: Cleanup & Closure (Week 8)
**Goal:** Remove technical debt and preventing rollback.

*   **Week 8: Cleanup**
    *   [Backend] Remove Dual-Write logic (Write ONLY to V3).
    *   [Backend] Remove Legacy Endpoints.
    *   [DBA] Archive & Drop Legacy Tables (`task`, `spraying`).

**Milestone 4:** Project Complete.

---

## 4. Risk Management

| Risk | Probability | Impact | Mitigation Strategy |
| :--- | :--- | :--- | :--- |
| **Data Corruption** | Medium | Critical | **Dual-Write** strategy ensures we always have the Legacy data as backup. We don't touch Legacy data, only read it. |
| **Performance Dip** | High | High | Indexing strategy (`idgroup` partition). Stress test the V3 Read Endpoint in Week 3. |
| **Team Delays** | High | Medium | The timeline is **8 weeks** (padded from an ideal 4-5 weeks). Weekly "Go/No-Go" checks. |
| **Migration Lock** | Medium | High | Migration script uses `INSERT IGNORE` and small chunks (1000) to prevent table locks. |

---

## 5. Definition of Done (DoD)
1.  **Parity:** `COUNT(legacy_tasks) == COUNT(new_tasks)`.
2.  **Performance:** Calendar load time < 500ms for 500 tasks.
3.  **Clean Code:** No `merge_key` logic in active code paths.
4.  **No Bugs:** Critical flows (Create, Update, Delete) pass regression.
