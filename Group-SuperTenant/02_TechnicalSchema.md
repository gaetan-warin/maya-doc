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

## 3. Role-Based Access Control (RBAC) Schema

Permissions are applied to Users within the context of a Tenant.

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│    user     │      │    roles    │      │ permissions │
│─────────────│      │─────────────│      │─────────────│
│ iduser      │◄──┐  │ idrole      │◄──┐  │ idpermission│
└─────────────┘   │  │ name        │   │  │ slug        │
                  │  └─────────────┘   │  └─────────────┘
                  │                    │
           ┌──────┴──────┐      ┌──────┴──────┐
           │ role_user   │      │role_permiss…│
           │ (Implicit)  │      │─────────────│
           │ iduser      │      │ idrole      │
           │ idrole      │      │ idpermission│
           │ idtenant    │      └─────────────┘
           └─────────────┘
```

### 3.1 `roles`
Definitions of role types (e.g., "Admin", "Greenkeeper").
*   Managed via `RoleController`.

### 3.2 `permissions`
Granular capabilities (e.g., `task.create`, `report.view`).

### 3.3 `role_permissions`
Links Roles to Permissions. allowing flexible RBAC updates without changing code.

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
