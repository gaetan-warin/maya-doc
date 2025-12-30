# Group Permission Deep Dive
## Creation, Verification, and Updates

| Document Details | |
| :--- | :--- |
| **Project** | Maya Core 2.0 |
| **Module** | Identity & Access Management |
| **Topic** | Group Permissions Implementation |

---

## 1. Overview
Group Permissions in Maya are implemented using a **Materialized Template Pattern**. Instead of calculating permissions dynamically via joins every time, specific boolean flags are stored in the `group_permission` table for each user.

### Key Components
*   **Service:** `App\Services\GroupPermissionService`
*   **Repository:** `App\Repositories\GroupPermissionRepository`
*   **Model:** `App\Models\GroupPermission`
*   **Helper:** `App\Helpers\GroupPermissionHelper` (Defines the columns)

---

## 2. Creation Workflow

**Trigger:** New User Creation or User Role Change.

### Step 1: Template Lookup
The system determines the user's role (e.g., `'admin'`, `'greenkeeper'`) and fetches the corresponding template from `group_permission_template`.

```php
// App\Services\GroupPermissionService.php
$groupPermissionTemplate = $this->groupPermissionTemplateService->getGroupPermissionTemplate(
    $data['user_role'], 
    $user->idtenant
);
```

### Step 2: Permission Generation
The service merges the template's boolean flags with the User ID and Group ID.

```php
// App\Repositories\GroupPermissionRepository.php
public function createGroupPermission(User $user, array $groupPermissionTemplate): ?GroupPermission
{
    $basicDataArray = [
        'idgroup_permission' => $this->generateUniqueUUID(...),
        'idgroup'            => $user->iddefault_group,
        'iduser'             => $user->iduser,
    ];
    $data = array_merge($groupPermissionTemplate, $basicDataArray);
    return $this->groupPermissionModel->create($data);
}
```

**Result:** A new row in `group_permission` containing explicit `1` or `0` for every capability (e.g., `task_create=1`, `spraying_delete=0`).

---

## 3. Verification Workflow

**Trigger:** API Request or Business Logic Check.

### Mechanism: Direct Column Check
There is no RBAC traversal. The check is a simple `WHERE` clause on the user's permission row.

```php
// App\Repositories\GroupPermissionRepository.php
public function hasGroupPermission(string $permission, string $userId): bool
{
    // SELECT exists(1) FROM group_permission 
    // WHERE iduser = uuid(?) AND task_create = 1
    return $this->groupPermissionModel
        ->where([
            'iduser' => $this->convertUuidWhereTo($userId),
            $permission => 1
        ])->exists();
}
```

### List of Verifiable Permissions
All available permission columns are defined in `App\Helpers\GroupPermissionHelper::getGroupPermissionColumns()`.
Examples include:
*   `task_create`, `task_read`, `task_update`, `task_delete`
*   `spraying_create`, `spraying_read`
*   `reports_create`, `reports_read`
*   `user_create`, `user_delete`

---

## 4. Update Workflow

**Trigger:** User Role Update (`GroupPermissionService::updateGroupPermission`).

### Strategy: Replace on Change
The system does not support "delta updates" (e.g., "add just this one permission"). It treats the Role as the source of truth.

1.  **Check:** Did the `user_role` change?
2.  **Delete:** If yes, call `deleteByUserId($userId)` to remove the old permission row.
3.  **Create:** Call `createGroupPermission` to generate a fresh row from the *new* template.

```php
// App\Services\GroupPermissionService.php
if ($user->user_role !== $data['user_role']) {
    $this->deleteByUserId($userId);
    return $this->createGroupPermission($user, $data);
}
```

### Implications
*   **Customization:** You cannot easily give a specific user "Admin + 1 extra permission" unless you create a new Custom Template/Role.
*   **Performance:** extremely fast reads (0 joins), but slightly heavier writes (row replacement).
