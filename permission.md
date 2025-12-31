# Permission Store Guide

## Overview

The Permission Store is a centralized Pinia store that manages all user permissions in the Maya application. It provides a secure, backend-driven permission system with role-based access control (RBAC) and package-level subscription features.

---

## 🎯 Key Concepts

### Permission Types

The system supports **4 types of permissions**:

| Type | Description | Admin Bypass |
|------|-------------|--------------|
| **Package Permissions** | Tenant-level subscription features (e.g., access to Dashboard, Weather, Inventory modules) | ❌ **NO** - Subscription level |
| **Role Permissions** | Feature-level toggles (e.g., manage_schedule, manage_stock) | ✅ YES - Owner/Admin bypass |
| **Group Permissions** | Entity CRUD operations (e.g., CREATE, READ, UPDATE, DELETE for Products, Users, etc.) | ✅ YES - Owner/Admin bypass |
| **Inventory Roles** | Special inventory-specific permissions that override entity permissions | ⚠️ Partial - Role-specific |

### Permission Priority

When checking permissions, the system follows this priority order:

```
1. Package Permissions (checked first, NOT bypassed by admin)
   ↓
2. Admin Bypass (Owner/Admin skip remaining checks)
   ↓
3. Entity CRUD Permissions (group_permissions)
   ↓
4. Role Permissions (role_permissions array)
   ↓
5. Settings Permissions
```

---

## 📦 Installation & Setup

### 1. Import the Store

```javascript
// Composition API
import { usePermissionStore } from '@/store/permission'

const permissionStore = usePermissionStore()
```

```javascript
// Options API - Already available globally (Legacy support, should be remove)
this.$hasPermission(...)
this.$PERMISSIONS
this.$ENTITIES
this.$CRUD_ACTIONS
this.$ROLES
```

### 2. Using the Composable (Recommended)

```javascript
import { usePermissions } from '@/composables/usePermissions'

const {
  hasPermission,
  PERMISSIONS,
  ENTITIES,
  CRUD_ACTIONS,
  ROLES
} = usePermissions()
```

---

## 🔐 Checking Permissions

### Basic Permission Check

```javascript
// Check a single permission
if (hasPermission(PERMISSIONS.SCHEDULE.MANAGE)) {
  // User can manage schedules
}

// Check package permission
if (hasPermission(PERMISSIONS.PACKAGE.WEATHER)) {
  // User's tenant has weather module access
}
```

### Entity CRUD Check

```javascript
// Check if user can CREATE products
if (hasPermission(ENTITIES.PRODUCT, CRUD_ACTIONS.CREATE)) {
  // Show "Add Product" button
}

// Using ROLES shorthand
if (ROLES.PRODUCT.CREATE) {
  // Same as above - more concise
}
```

### Multiple Permissions (OR Logic)

```javascript
// Check if user has ANY of these permissions
const canAccessPurchase = hasPermission([
  PERMISSIONS.PURCHASE.PLACE_REQUEST,
  PERMISSIONS.PURCHASE.MANAGE_ORDER,
  PERMISSIONS.PURCHASE.MANAGE_INVOICES
], undefined, 'OR') // OR is default
```

### Multiple Permissions (AND Logic)

```javascript
// Check if user has ALL of these permissions
const canFullyManageFleet = hasPermission([
  PERMISSIONS.FLEET.ADD_NEW_MACHINE,
  PERMISSIONS.FLEET.EDIT_MACHINERY_PROPERTIES,
  PERMISSIONS.FLEET.MANAGE_MAINTENANCE_SCHEDULES
], undefined, 'AND')
```

### Page Access Check

```javascript
// Check if user can access a specific page
// Note: Route names come from your router configuration (to.name)
// This example shows the value for demonstration purposes
const routeName = 'fleet_management' // From route object: to.name
if (permissionStore.hasPageAccess(routeName)) {
  // Render fleet management page
}

// In router guard - use route object directly
if (!permissionStore.hasPageAccess(to.name)) {
  return '/dashboard' // Redirect
}
```

---

## 🎨 Template Usage

### Vue Template with v-if

```vue
<template>
  <!-- Using composable -->
  <button v-if="hasPermission(PERMISSIONS.PRODUCT.ADD_CREATE_EDIT)">
    Add Product
  </button>

  <!-- Using ROLES shorthand -->
  <button v-if="ROLES.PRODUCT.CREATE">
    Create Product
  </button>

  <!-- Options API -->
  <button v-if="$hasPermission($PERMISSIONS.SCHEDULE.MANAGE)">
    Manage Schedule
  </button>

  <!-- Multiple permissions -->
  <div v-if="hasPermission([
    PERMISSIONS.STOCK.MANAGE,
    PERMISSIONS.STOCK.MANAGE_DELIVERIES
  ], undefined, 'OR')">
    Stock Management Section
  </div>
</template>

<script setup>
import { usePermissions } from '@/composables/usePermissions'

const { hasPermission, PERMISSIONS, ROLES } = usePermissions()
</script>
```

### Computed Properties

```vue
<script setup>
import { computed } from 'vue'
import { usePermissions } from '@/composables/usePermissions'

const { hasPermission, PERMISSIONS, ROLES } = usePermissions()

const canManageStock = computed(() =>
  hasPermission(PERMISSIONS.STOCK.MANAGE)
)

const canCreateProduct = computed(() =>
  ROLES.PRODUCT.CREATE
)

const showInventorySection = computed(() =>
  hasPermission([
    PERMISSIONS.STOCK.MANAGE,
    PERMISSIONS.PURCHASE.MANAGE_ORDER
  ], undefined, 'OR')
)
</script>

<template>
  <div v-if="canManageStock">
    <ProductList />
  </div>

  <button v-if="canCreateProduct">
    Add New Product
  </button>
</template>
```

---

## 🔑 Available Permission Constants

### PERMISSIONS Object Structure

```javascript
PERMISSIONS = {
  // Schedule & Planning
  SCHEDULE: {
    MANAGE: 'manage_schedule',
    ACCESS_PAGE_AND_JOBS: 'access_page_and_jobs',
    ACCESS_PLANNER_SECTION: 'access_planner_section',
    ACCESS_PAGE: 'access_schedule_page',
    MANAGE_MY_OWN_TASK: 'manage_my_own_task',
  },

  // Inventory & Stock
  STOCK: {
    MANAGE: 'manage_stock',
    MANAGE_DELIVERIES: 'manage_deliveries',
    RECEIVE_LOW_STOCK_LEVEL_ALERT: 'receive_low_stock_level_alert',
    MANAGE_PRODUCTS_CATEGORIES: 'manage_stock_products_categories',
    MANAGE_MACHINE_CATEGORIES: 'manage_stock_machine_categories',
  },

  // Purchase & Finance
  PURCHASE: {
    PLACE_REQUEST: 'place_purchase_request',
    MANAGE_ORDER: 'manage_purchase_order',
    VALIDATE_MANAGE_REQUEST: 'validate_manage_purchase_request',
    MANAGE_INVOICES: 'manage_invoices',
    MANAGE_PAYMENT_STATUS: 'manage_payment_status',
    MANAGE_VAT_AND_TAX: 'manage_vat_and_tax_rates',
  },

  // Products
  PRODUCT: {
    ADD_CREATE_EDIT: 'add_create_edit_product',
  },

  // Fleet & Machinery
  FLEET: {
    UPDATE_MACHINERY_HOURS: 'update_machinery_hours',
    UPDATE_FUEL_LEVEL: 'update_fuel_level',
    ADD_NEW_MACHINE: 'add_new_machine',
    MANAGE_MAINTENANCE_SCHEDULES: 'manage_maintenance_schedules',
    ADD_NEW_MAINTENANCE: 'add_new_maintenance',
    EDIT_MACHINERY_PROPERTIES: 'edit_machinery_properties',
    TRANSFER_MACHINE_BETWEEN_TENANTS: 'transfer_machine_between_tenants',
    ADD_MACHINE_HOURS: 'add_machine_hours',
  },

  // Package Permissions (Subscription Level)
  PACKAGE: {
    DASHBOARD: 'dashboard_access',
    WEATHER: 'weather_access',
    GRAPH_BUILDER: 'graph_builder_access',
    INSIGHT: 'insight_access',
    ALARMS: 'alarms_access',
    HISTORY: 'history_access',
    FORECAST: 'forecast_access',
    SCHEDULE: 'schedule_access',
    MAPS: 'maps_access',
    INVENTORY: 'inventory_access',
    REPORTS: 'reports_access',
    WATER: 'water_access',
    SOIL_ANALYSIS: 'soil_analysis_access',
    SPRAYING: 'spraying_access',
    FLEET_MANAGEMENT: 'fleet_management_access',
    SETTING: 'setting_access',
    GRASS_ROOT: 'grass_root_access',
    BUDGET_TOOL: 'budget_tool_access',
    FERTILISATION_PLANNER: 'fertilisation_planner',
    BUDGET_PLANNER: 'budget_planner',
    DAILY_ROUTINE: 'daily_routine',
    SPRAYING_ROUTINE: 'spraying_routine',
    ACTION_PLANNER: 'action_planner',
    INTERACTIVE_REPORT: 'interactive_report',
    USER_MANAGEMENT: 'user_management',
  },

  // ... (see constants.ts for complete list)
}
```

### ENTITIES & CRUD_ACTIONS

```javascript
// Entity types
ENTITIES = {
  PURCHASE: 'PURCHASE',
  SUPPLIER: 'SUPPLIER',
  PRICE: 'PRICE',
  PRODUCT: 'PRODUCT',
  TENANT: 'TENANT',
  SITE: 'SITE',
  SUBSITE: 'SUBSITE',
  DEVICE: 'DEVICE',
  USER: 'USER',
  SPRAYING: 'SPRAYING',
  REPORTS: 'REPORTS',
  ALARMS: 'ALARMS',
  TASK_SCHEDULES: 'TASK_SCHEDULES',
  TASK: 'TASK',
  STAFF: 'STAFF',
  MACHINE: 'MACHINE',
  SPRAYING_NOTIFICATION: 'SPRAYING_NOTIFICATION',
  FLEET_MANAGEMENT: 'FLEET_MANAGEMENT'
}

// CRUD actions
CRUD_ACTIONS = {
  CREATE: 'CREATE',
  READ: 'READ',
  UPDATE: 'UPDATE',
  DELETE: 'DELETE',
  IMPORT: 'IMPORT'
}
```

### ROLES Object (Computed)

```javascript
// Access CRUD permissions via ROLES computed object
ROLES = {
  PRODUCT: {
    CREATE: boolean,
    READ: boolean,
    UPDATE: boolean,
    DELETE: boolean
  },
  PURCHASE: {
    CREATE: boolean,
    READ: boolean,
    UPDATE: boolean,
    DELETE: boolean,
    IMPORT: boolean
  },
  // ... all entities
}
```

---

## 👥 User Roles

### Role Hierarchy

```javascript
USER_ROLES = {
  OWNER: 0,        // Full access (bypasses role/group permissions)
  ADMIN: 1,        // Full access (bypasses role/group permissions)
  EDITOR: 2,       // Limited by permissions
  VIEWER: 3,       // Limited by permissions
  TEAM_MANAGER: 4  // Special role with admin-like access to certain features
}
```

### Checking User Role

```javascript
import { usePermissionStore } from '@/store/permission'

const permissionStore = usePermissionStore()

// Check specific role
if (permissionStore.isOwner) {
  // User is owner
}

if (permissionStore.isAdmin) {
  // User is admin
}

if (permissionStore.isOwnerOrAdmin) {
  // User is owner OR admin
}

if (permissionStore.isEditor) {
  // User is editor
}

if (permissionStore.isViewer) {
  // User is viewer
}

// Using helper function
if (permissionStore.isAdminUser()) {
  // User has admin privileges
}
```

---

## 📋 Common Patterns

### 1. Conditional Rendering

```vue
<template>
  <!-- Show button only if user can create -->
  <button
    v-if="ROLES.PRODUCT.CREATE"
    @click="createProduct"
  >
    Create Product
  </button>

  <!-- Show section only if user has package access -->
  <section v-if="hasPermission(PERMISSIONS.PACKAGE.WEATHER)">
    <WeatherWidget />
  </section>

  <!-- Show for admin only -->
  <div v-if="permissionStore.isOwnerOrAdmin">
    <AdminPanel />
  </div>
</template>
```

### 2. Router Guards

```javascript
// In router/index.js
router.beforeEach(async (to, from, next) => {
  const permissionStore = usePermissionStore()

  // Check page access
  if (!permissionStore.hasPageAccess(to.name)) {
    return next('/dashboard')
  }

  // Check specific permission for route
  const requiredPermission = ROUTE_PERMISSIONS[to.name]
  if (requiredPermission && !permissionStore.hasPermission(requiredPermission)) {
    return next('/dashboard')
  }

  next()
})
```

### 3. API Call Guards

```javascript
async function deleteProduct(productId) {
  // Check permission before API call
  if (!ROLES.PRODUCT.DELETE) {
    toast.error('You do not have permission to delete products')
    return
  }

  try {
    await api.delete(`/products/${productId}`)
    toast.success('Product deleted')
  } catch (error) {
    toast.error('Failed to delete product')
  }
}
```

### 4. Dynamic Menu Items

```javascript
const menuItems = computed(() => {
  const items = []

  if (hasPermission(PERMISSIONS.SCHEDULE.MANAGE)) {
    items.push({ name: 'Schedule', route: '/schedule' })
  }

  if (hasPermission(PERMISSIONS.STOCK.MANAGE)) {
    items.push({ name: 'Stock', route: '/stock' })
  }

  if (hasPermission(PERMISSIONS.PACKAGE.FLEET_MANAGEMENT)) {
    items.push({ name: 'Fleet', route: '/fleet_management' })
  }

  return items
})
```

### 5. Form Field Permissions

```vue
<template>
  <form @submit="saveProduct">
    <!-- Always visible -->
    <input v-model="product.name" placeholder="Product Name" />

    <!-- Only editable if user can update -->
    <input
      v-model="product.price"
      placeholder="Price"
      :disabled="!ROLES.PRODUCT.UPDATE"
    />

    <!-- Only visible if user can manage prices -->
    <input
      v-if="ROLES.PRICE.UPDATE"
      v-model="product.costPrice"
      placeholder="Cost Price"
    />

    <!-- Submit button with permission check -->
    <button
      type="submit"
      :disabled="!ROLES.PRODUCT.UPDATE"
    >
      Save Product
    </button>
  </form>
</template>
```

---

## 🔄 Refreshing Permissions

### Automatic Refresh

Permissions are automatically refreshed in these scenarios:

1. **On Login** - After successful authentication
2. **Token Refresh** - When access token is renewed by axios interceptor
3. **App Load** - On initial app bootstrap (if authenticated)

### Manual Refresh

```javascript
import { usePermissionStore } from '@/store/permission'

const permissionStore = usePermissionStore()

// Manually refresh permissions from backend
try {
  await permissionStore.refreshPermissions()
  console.log('Permissions refreshed')
} catch (error) {
  console.error('Failed to refresh permissions:', error)
}
```

---

## ⚙️ Advanced Usage

### Inventory Role Overrides

Inventory roles provide special permission overrides for inventory-related entities:

```javascript
INVENTORY_ROLES = {
  DENIED: 0,              // No inventory access
  ACCOUNTANT: 1,          // Cannot CREATE purchases
  FINANCIAL_DIRECTOR: 2,  // Full PURCHASE access
  STOCK_MANAGER: 3,       // Full PRODUCT, PRICE, SUPPLIER access
  FLEET_MANAGER: 4,       // Fleet-specific access
  FLEET_ADMIN: 5          // Full fleet access
}
```

Example:
```javascript
// User with ACCOUNTANT inventory role
// Even if group_permissions.statement_create = 1
// They CANNOT create purchases due to inventory role override

if (ROLES.PURCHASE.CREATE) {
  // This will be false for ACCOUNTANT role
  // even if group permission allows it
}
```

### Admin-Only Pages

Some pages require owner, admin, or team manager access:

```javascript
ADMIN_ONLY_PAGES = [
  'graphbuilder',
  'soil_analysis',
  'budget_tool',
  'history'
]

// Router guard checks
if (ADMIN_ONLY_PAGES.includes(routeName) && !permissionStore.isOwnerOrAdmin) {
  return '/dashboard'
}
```

### SSO Permission Check

```javascript
import { usePermissionStore } from '@/store/permission'

const permissionStore = usePermissionStore()

if (permissionStore.hasSsoPermission()) {
  // Show SSO login option
}
```

---

## 🚨 Important Notes

### ⚠️ Package Permissions ARE NOT Bypassed by Admin

```javascript
// ❌ WRONG ASSUMPTION
// "I'm an admin, so I can access all modules"

// ✅ CORRECT
// Package permissions are tenant-level subscription features
// Even owners/admins cannot bypass them

if (hasPermission(PERMISSIONS.PACKAGE.WEATHER)) {
  // This checks the TENANT's subscription
  // NOT the user's role
}
```

### ⚠️ Never Use localStorage Directly

```javascript
// ❌ BAD - Don't do this
const userInfo = JSON.parse(localStorage.getItem('userInfo'))
if (userInfo.role_permissions.includes('manage_schedule')) {
  // This is insecure and can be outdated
}

// ✅ GOOD - Use the permission store
if (hasPermission(PERMISSIONS.SCHEDULE.MANAGE)) {
  // This checks the backend-validated permissions
}
```

### ⚠️ Always Check Permissions on Backend

```javascript
// Frontend permission checks are for UI only
// Always validate permissions on the backend API

// Frontend (UI control)
if (ROLES.PRODUCT.DELETE) {
  // Show delete button
}

// Backend must also validate
// DELETE /api/products/:id
// → Check user permissions before deleting
```

---

## 🐛 Debugging

### Check Permission State

```javascript
import { usePermissionStore } from '@/store/permission'

const permissionStore = usePermissionStore()

// Check if permissions are initialized
console.log('Initialized:', permissionStore.isInitialized)

// Check if currently loading
console.log('Loading:', permissionStore.isLoading)

// Check for errors
console.log('Error:', permissionStore.error)

// View all permissions
console.log('Permissions:', permissionStore.permissions)

// View user role
console.log('User Role:', permissionStore.permissions.userRole)

// View role permissions
console.log('Role Permissions:', permissionStore.permissions.rolePermissions)

// View package permissions
console.log('Package Permissions:', permissionStore.permissions.packagePermissions)
```

### Console Logging

```javascript
// Enable detailed logging (development only)
const hasAccess = hasPermission(PERMISSIONS.SCHEDULE.MANAGE)
console.log('Has schedule permission:', hasAccess)

// Check why permission is denied
console.log('Package:', permissionStore.permissions.packagePermissions[PERMISSIONS.PACKAGE.SCHEDULE])
console.log('Role:', permissionStore.isOwnerOrAdmin)
console.log('Role Perms:', permissionStore.permissions.rolePermissions)
```

---

## 📚 Related Files

- **Store**: `src/store/permission/store.ts`
- **Logic**: `src/store/permission/logic.ts`
- **API**: `src/store/permission/api.ts`
- **Constants**: `src/store/permission/constants.ts`
- **Types**: `src/store/permission/types.ts`
- **Composable**: `src/composables/usePermissions.ts`
- **Plugin**: `src/plugins/permissionPlugin.ts`

---

## 🎓 Best Practices

1. **Use Constants**: Always use `PERMISSIONS`, `ENTITIES`, `CRUD_ACTIONS` constants instead of strings
2. **Use Composable**: Prefer `usePermissions()` composable over direct store access
3. **Use ROLES Shorthand**: For CRUD checks, use `ROLES.ENTITY.ACTION` for cleaner code
4. **Check Early**: Check permissions before rendering components or making API calls
5. **Backend Validation**: Always validate permissions on backend, frontend checks are UI-only
6. **Computed Properties**: Use computed properties for reactive permission checks
7. **Avoid localStorage**: Never check permissions directly from localStorage
8. **Package vs Role**: Remember package permissions are subscription-level, not role-level

---

## 💡 Quick Reference

```javascript
// Import
import { usePermissions } from '@/composables/usePermissions'
const { hasPermission, PERMISSIONS, ENTITIES, CRUD_ACTIONS, ROLES } = usePermissions()

// Single permission
hasPermission(PERMISSIONS.SCHEDULE.MANAGE)

// CRUD permission
hasPermission(ENTITIES.PRODUCT, CRUD_ACTIONS.CREATE)
ROLES.PRODUCT.CREATE // Shorthand

// Multiple permissions (OR)
hasPermission([PERMISSIONS.STOCK.MANAGE, PERMISSIONS.PURCHASE.MANAGE_ORDER], undefined, 'OR')

// Multiple permissions (AND)
hasPermission([...permissions], undefined, 'AND')

// Page access
const routeName = to.name // Get from route object
permissionStore.hasPageAccess(routeName)

// Role checks
permissionStore.isOwner
permissionStore.isAdmin
permissionStore.isOwnerOrAdmin
permissionStore.isEditor
permissionStore.isViewer

// Admin check
permissionStore.isAdminUser()

// SSO check
permissionStore.hasSsoPermission()
```

---

**Need Help?** Check the source code in `src/store/permission/` or reach out to the development team.
