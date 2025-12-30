# Access Control & Permissions Flow
## Identity Management Guide

| Document Details | |
| :--- | :--- |
| **Project** | Maya Core 2.0 |
| **Scope** | Authentication & Authorization |

---

## 1. Authentication Flow

### 1.1 Standard Login
1.  User posts credentials to `/api/auth/login`.
2.  System verifies password.
3.  System loads the User's primary **Tenant** and **Default Group**.
4.  **Token Issuance:** Access Token implies context of `(User + Tenant + Group)`.

### 1.2 Super Tenant Logic
1.  Super Admin authenticates against the Super Tenant dashboard.
2.  They select a specific Tenant from the list (via `getSuperTenantListByTenant`).
3.  **Context Construction:** The API likely treats this as a scoped request, validating that `Link(SuperTenant -> Tenant)` exists before returning data.

---

## 2. Authorization (Permissions)

### 2.1 The Triangle of Power
Access is determined by the intersection of:
1.  **Who you are** (User)
2.  **Where you are** (Group Context)
3.  **What role you hold** (Template)

### 2.2 Template-Based Model
Unlike systems where permissions are calculated at runtime (`User -> Role -> Permissions`), Maya uses a **Copy-on-Assignment** model.
1.  **Templates:** Defined in `group_permission_template` (e.g., "Admin", "Greenkeeper").
2.  **Assignment:** When a User is created/updated, they are assigned a `user_role`.
3.  **Materialization:** The system looks up the Template and **copies** the permission flags to a new row in `group_permission` linked to the user.

### 2.3 Checking Permissions in Code

**Do not check Roles.** Always check **Permissions**.

❌ **Bad:**
```php
if ($user->role->name === 'Admin') { // brittle }
```

✅ **Good:**
```php
// Uses GroupPermissionRepository::hasGroupPermission
if ($this->groupPermissionService->hasGroupPermission('task_delete', $user->iduser)) {
   // proceed
}
```

This allows granular control. For example, you could theoretically toggle `task_delete` to `false` for a specific Admin user in the database without changing their Role, although the current `update` logic resets this from the template.

---

## 3. Partners & External Access

### 3.1 Partner Tenants
Partners (`partners` table) are external entities (like agronomists or consultants) who need access to a Tenant.
*   Managed via `PartnerController`.
*   Linkage: `partner_tenant` table.
*   Provides restricted, scoped access compared to full internal users.

---

## 4. Best Practices for Developers

1.  **Never assume Tenant = Group.** Always verify if the feature is "Global" (Tenant) or "Site Specific" (Group).
2.  **Scope all queries.** `Where('idgroup', $currentGroupId)`.
3.  **Use Middleware.** Ensure routes are protected by `auth:api` and potentially valid-tenant middleware.
4.  **UUID Casting.** Remember that the DB stores Binary UUIDs. Use traits like `UuidHelper` to convert to/from Strings for the API.
