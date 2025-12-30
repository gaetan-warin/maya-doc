# Technical Schema Reference
## Identity & Hierarchy Data Model

| Document Details | |
| :--- | :--- |
| **Project** | Maya Core 2.0 |
| **Scope** | Database Schema for Isolation |

---

## 1. Entity Relationship Diagram (ERD)

```
┌─────────────────┐       ┌─────────────────┐
│  super_tenant   │       │     tenant      │
│ (Aggregator)    │       │ (Customer)      │
│─────────────────│       │─────────────────│
│ idsuper_tenant  │◄───┐  │ idtenant (UUID) │◄──┐
│ name            │    │  │ iddefault_group │   │
└─────────────────┘    │  │ name            │   │
                       │  └────────┬────────┘   │
                       │           │            │
┌─────────────────┐    │           │ 1          │ 1
│super_tenant_link│    │           │            │
│─────────────────│    │           ▼ *          │
│ idsuper_tenant  │────┘  ┌─────────────────┐   │
│ idtenant (UUID) │       │      group      │   │
└─────────────────┘       │ (Op Unit)       │   │
                          │─────────────────│   │
                          │ idgroup (UUID)  │   │
                          │ idtenant (FK)   │───┘
                          │ name            │
                          └─────────────────┘
```

---

## 2. Table Definitions

### 2.1 `tenant`
The root billing and configuration entity.

| Column | Type | Key | Description |
| :--- | :--- | :--- | :--- |
| `idtenant` | BINARY(16) | PK | UUID primary key. |
| `iddefault_group` | BINARY(16) | FK | The default operational unit for this tenant. |
| `name` | VARCHAR | | Tenant display name. |
| `sso_enabled` | BOOLEAN | | Security configuration. |

### 2.2 `group`
The operational partition. **Most data queries join here.**
*   *Note: In the codebase, this is handled by `App\Models\Group`.*

| Column | Type | Key | Description |
| :--- | :--- | :--- | :--- |
| `idgroup` | BINARY(16) | PK | UUID primary key. |
| `idtenant` | BINARY(16) | FK | Parent tenant. |
| `name` | VARCHAR | | Group display name (e.g., "North Course"). |

### 2.3 `super_tenant`
A lightweight aggregation container.

| Column | Type | Key | Description |
| :--- | :--- | :--- | :--- |
| `idsuper_tenant` | BIGINT | PK | Auto-increment ID (Legacy pattern mixed with new). |
| `name` | VARCHAR | | Display name (e.g., "Global Distributor"). |

### 2.4 `super_tenant_link`
Junction table mapping Tenants to Super Tenants.

| Column | Type | Key | Description |
| :--- | :--- | :--- | :--- |
| `idsuper_tenant` | BIGINT | FK | Parent Super Tenant. |
| `idtenant` | CHAR(16) | FK | Child Tenant (Stored as Binary in standard tables, verify format). |

---

## 3. Permission Schema

### 3.1 `group_permission`
The materialized permission set for a specific user in a specific group.
*   **Key Concept:** Permissions are **flattened** here. We don't join to roles at runtime.

| Column | Type | Key | Description |
| :--- | :--- | :--- | :--- |
| `idgroup_permission` | BINARY(16) | PK | UUID. |
| `idgroup` | BINARY(16) | FK | The group context. |
| `iduser` | BINARY(16) | FK | The user. |
| `task_create` | BOOLEAN | | Specific capability. |
| `task_read` | BOOLEAN | | Specific capability. |
| `...` | ... | | (See `GroupPermissionHelper.php` for full list). |

### 3.2 `group_permission_template`
The "Role Definition" that populates `group_permission`.

| Column | Type | Description |
| :--- | :--- | :--- |
| `idgroup_permission_template` | PK | UUID. |
| `role_name` | String | e.g. "Admin", "Greenkeeper", "Assistant". |
| `task_create` | BOOLEAN | Template value. |
| `...` | ... | |

---

## 4. Key Considerations for Developers

### 4.1 UUIDs vs Integers
*   **Tenants and Groups** use **UUIDs** (Binary 16).
*   **Super Tenants** use **BigInt**.
*   *Be careful when joining or casting types in API responses.*

### 4.2 The "Group Context"
When writing queries for `Tasks`, `Spraying`, or `Checklists`, you must technically filter by `idgroup`.
*   Filtering by `idtenant` is often insufficient because a Tenant might have multiple Groups (Courses), and mixing their data is a data leak in multi-site operations.

### 4.3 Super Tenant Access
A Super Tenant admin does **not** have a direct login to the Tenant's data. They likely use an "impersonation" or "context switch" flow handled by the `SuperTenantController` to obtain a scoped token.
