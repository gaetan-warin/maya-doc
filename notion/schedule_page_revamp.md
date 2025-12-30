# Schedule Page Revamp — Unified Vision, Phased Execution

## 1. Why We Are Doing This (Problem Statement)

The Schedule module is one of the most **business-critical parts of Maya**. It is used daily across golf, football and multisport environments to plan, execute and analyse operational work.

### Current Issues:

| Problem | Impact |
|---------|--------|
| Accumulated technical debt | Fragile, slow UX |
| Fragmented API contracts | 20+ API calls per page load |
| Inconsistent data models | Backend contracts not fit for purpose |
| Legacy table structure | Tasks, labour, nutritional inputs split across tables with overlapping responsibility |

> A full rewrite of FE, BE and DB at once would be high-risk and destabilising. We are taking a **deliberate, phased approach**.

---

## 2. The Guiding Principle

> **Stabilise what users touch first. Fix structure once behaviour is stable.**

This plan explicitly separates:
- **Short-term:** Stabilisation & UX improvement
- **Long-term:** Structural refactor

Both are necessary — but not at the same time.

---

## 3. Phase 1 — Stabilisation & UX-Driven Refactor (Immediate Priority)

### Objective
Deliver a fast, stable, scalable Schedule page that:
- Supports large teams smoothly
- Implements correct business logic (planned vs actual, multi-site, overtime)
- Reduces API calls dramatically (v1.0)
- **Does NOT change the underlying database schema yet**

*Aligns with Epic 242*

### 3.1 Frontend Scope (Owner: Simón)

**Key Outcomes:**
- New tab-based Schedule UI:
  - Calendar Week
  - Day Plan (per-staff Gantt)
  - Site Overview (per pitch/course/zone)
- Unified task wizard with:
  - Smart defaults
  - Conflict visibility
  - Improved duration handling (multi-staff, multi-site)
- UI aligned with TO-BE business logic:
  - One task = one planned duration
  - Planned labour = duration × staff
  - Multi-site splitting rules
  - Planned vs actual comparison (mobile tracking preserved)

**Critical Deliverable by 5 January:**
API contract definition describing:
- Required payloads
- Aggregation expectations
- Performance assumptions

### 3.2 Backend Scope (Owner: Dilanka)

**Key Objectives:**
- Collapse 20+ API calls into domain-oriented endpoints
- Align backend responses with actual UI needs
- Optimise queries within existing schema
- Formalise clean FE ↔ BE contract

**Constraints:**
- No database refactor in Phase 1
- No behavioural change to mobile tracking or labour accounting
- Backward compatibility preserved

**Timeline:**
- January: Implement backend changes based on Simon's contract
- February: Integration and stabilisation with new UI

---

## 4. Phase 2 — Database & Domain Model Refactor (Parallel Design, Deferred Execution)

### Why Separate?
The Schedule table is:
- Heavily reused across the app
- Coupled to stock, reporting, mobile logs, routines, etc.

Refactoring blindly would create systemic risk.

### 4.1 DB Review & Target Architecture (Owner: Gaetan)

Over the next two weeks:
1. Analyse current Schedule-related tables
2. Identify inconsistencies and coupling
3. Propose target data model aligned with TO-BE logic:
   - Unified task + nutritional input concept
   - Correct labour accounting
   - Legal / H&S traceability
   - Future ML and reporting readiness

*This work is architectural, not immediately executable.*

### 4.2 Database Refactor — Timing & Decision Gate

#### Step 1 — Findings & Target Direction
- **Owner:** Gaetan
- **Timing:** Week of 5 January

Gaetan will present:
- Assessment of current Schedule-related table structure
- Identification of:
  - Structural inconsistencies
  - Redundancies / tight coupling
  - Misalignment with TO-BE business model
- Proposed target data model

*This step is diagnostic and architectural only. No refactoring or migration work starts.*

### 4.3 Migration Strategy (Defined Later)
- Likely approach: Parallel tables or compatibility layer
- Gradual switching by feature or context
- **No "big bang" migration**

---

## Timeline Summary

| Phase | Timeline | Ownership | Deliverable |
|-------|----------|-----------|-------------|
| Phase 1A | Dec 16 – Jan 5 | Simon (Belgium) | API Contract Specification |
| Phase 1B | Dec 16 – Dec 31 | Gaetan (Belgium) | DB Schema Analysis & TO-BE Proposal |
| Phase 2 | January 2025 | Simon: UI Build / Dilanka: API Refactor | New Vue.js Components + Optimised Laravel Endpoints |
| Phase 3 | February 2025 | Simon + Dilanka | Frontend-Backend Integration & QA |
| Phase 4 | Q1–Q2 2025 | TBD | Database Schema Migration (Parallel Operation) |

**Target Completion:** Summer 2026

---

(Archived from Notion on 2025-12-22)
