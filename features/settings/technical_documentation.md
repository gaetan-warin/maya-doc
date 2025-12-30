# Settings Module
## Technical Design Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-TSD-SETT-001 |
| **Version** | 1.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Engineering Team |
| **Classification** | Internal - Technical |

---

## 1. System Architecture

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Frontend (Vue 3)                          │
├─────────────────────────────────────────────────────────────────┤
│  Views           │  Components (25+)             │  Stores       │
│  └─ setting.vue  │  ├─ MyAccount.vue (33KB)     │  setting.js   │
│                  │  ├─ GeneralSetting.vue (22KB)│  (498 lines)  │
│                  │  ├─ ScheduleSetting.vue (23KB)│               │
│                  │  ├─ DashboardSetting.vue     │  settings/    │
│                  │  ├─ AgronomySetting.vue      │  └─ fleetTags │
│                  │  ├─ WaterSetting.vue         │               │
│                  │  └─ 19+ more components      │               │
├─────────────────────────────────────────────────────────────────┤
│                          API Layer                               │
│                    apiClient (Axios V2)                          │
├─────────────────────────────────────────────────────────────────┤
│                     Backend (Node.js)                            │
│     SettingService + ProfileService + DeviceService + TokenSvc  │
├─────────────────────────────────────────────────────────────────┤
│                      Database (MySQL)                            │
│   user_settings | tenant_config | device | api_token | site     │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Layer | Technology | Version |
|:---|:---|:---|
| Frontend Framework | Vue.js | 3.x |
| State Management | Pinia | 2.x |
| Maps | Google Maps API | - |
| Backend | Node.js / Express | 18.x |

---

## 2. Component Architecture

### 2.1 View Component

| Component | Path | Responsibility |
|:---|:---|:---|
| `setting.vue` | `views/` | Main container with tab navigation (364 lines) |

### 2.2 Setting Components (25+)

| Component | Size | Responsibility |
|:---|:---|:---|
| `MyAccount.vue` | 33KB | Profile, WhatsApp config |
| `GeneralSetting.vue` | 22KB | Units, notifications |
| `ScheduleSetting.vue` | 23KB | Mowing, leave, holidays |
| `DashboardSetting.vue` | 24KB | Widget configuration |
| `ApiTokenSetting.vue` | 15KB | Token management |
| `GetStartedSetting.vue` | 16KB | Onboarding wizard |
| `AgronomySetting.vue` | 12KB | Insight config |
| `ChangePassword.vue` | 9KB | Password update |
| `MyConnectedDevices.vue` | 8KB | Device management |
| `VWCSetting.vue` | 7KB | Soil sensor thresholds |
| `WaterSetting.vue` | 3KB | Water notifications |
| `FleetSetting.vue` | 5KB | Fleet options |
| `InventorySetting.vue` | 2KB | Stock preferences |
| `ReportSetting.vue` | 5KB | Report options |
| `AlarmSetting.vue` | 4KB | Alarm preferences |
| `SystemSetting.vue` | 3KB | System info |
| `FusariumSetting.vue` | 2KB | Disease threshold |
| `BussinessType.vue` | 1KB | Business display |

### 2.3 Subdirectories

| Directory | Purpose |
|:---|:---|
| `ApiToken/` | Token management components (8 files) |
| `FleetTag/` | Tag management (2 files) |
| `MowingDirection/` | Mowing config (3 files) |
| `water/` | Water source management (6 files) |
| `inventory/` | Inventory settings (11 files) |
| `leaveRequest/` | Leave type management (3 files) |
| `brand/` | Brand settings (4 files) |
| `agronomy/` | Agronomy config (5 files) |
| `grass-root/` | Grass root settings (1 file) |

---

## 3. State Management

### 3.1 Setting Store

**File**: `web/src/store/setting.js` (498 lines, 38 actions)

```javascript
// State Structure
{
  api: piniaEndPoint(),
  open: false,
  tenantId: null,
  lastOpenedTab: null,
  precisionTypes: [],
  nutritionalInputSetting: {},
  dashboardSetting: {},
  myProfile: {},
  devices: [],
  areaHoles: [],
  brands: [],
  fuelTypes: [],
  categoryTypes: [],
  stockNotificationSettings: {}
}
```

### 3.2 Key Store Actions (38 total)

| Category | Actions |
|:---|:---|
| **Agronomy** | `getPrecisionTypes`, `createPrecisionType`, `updatePrecisionType`, `deletePrecisionType` |
| **Nutritional** | `getNutritionalInputSettingData`, `putNutritionalInputSetting` |
| **Profile** | `editMyProfile`, `changePassword`, `getWhoAmI` |
| **Dashboard** | `saveDashboardSetting` |
| **Devices** | `getAllDevices` |
| **Areas** | `getAreaHoles`, `getAreaHolesSurface`, `upsertAreaHoles`, `saveTotalForArea` |
| **Brands** | `fetchBrands`, `postBrandRecord`, `fetchBrandById`, `deleteBrandRecord` |
| **Fleet** | `fetchFuelTypes`, `fetchCategoryTypes`, `getWhoAmIFleet` |
| **Maintenance** | `getMaintenanceByModelId`, `postMaintenance`, `deleteMaintenance` |
| **Stock** | `loadStockNotificationSettings`, `updateLowStockNotification`, `updateReminderFrequency` |

---

## 4. Data Models

### 4.1 User Settings Entity

**Table**: `user_settings`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `iduser` | CHAR(36) | User reference |
| `setting_key` | VARCHAR(100) | Setting identifier |
| `setting_value` | JSON | Configuration value |
| `updated_at` | TIMESTAMP | Last update |

### 4.2 Tenant Configuration Entity

**Table**: `tenant_config`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `idtenant` | CHAR(36) | Tenant reference |
| `measurement_unit` | ENUM | 'metric', 'imperial' |
| `weather_station_height` | DECIMAL(5,2) | Meters |
| `landscape_type` | VARCHAR(50) | Terrain type |
| `notification_channels` | JSON | {email, whatsapp, push} |

### 4.3 Device Entity

**Table**: `device`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `idtenant` | CHAR(36) | Tenant reference |
| `device_name` | VARCHAR(100) | Hardware ID |
| `display_name` | VARCHAR(100) | User label |
| `device_type` | ENUM | 'soil', 'weather', 'et' |
| `assigned_zone` | CHAR(36) | Zone reference |
| `status` | ENUM | 'active', 'partial', 'offline', 'voltage' |
| `last_seen` | TIMESTAMP | Last communication |

### 4.4 API Token Entity

**Table**: `api_token`

| Column | Type | Description |
|:---|:---|:---|
| `id` | CHAR(36) | UUID identifier |
| `idtenant` | CHAR(36) | Tenant reference |
| `name` | VARCHAR(100) | Token name |
| `token_hash` | VARCHAR(255) | Hashed token |
| `scope` | JSON | Permission scope |
| `created_at` | TIMESTAMP | Creation time |
| `expires_at` | TIMESTAMP | Expiration time |
| `status` | ENUM | 'active', 'revoked' |

### 4.5 Site Entity

**Table**: `site`

| Column | Type | Description |
|:---|:---|:---|
| `idsite` | CHAR(36) | UUID identifier |
| `idtenant` | CHAR(36) | Tenant reference |
| `name` | VARCHAR(100) | Site name |
| `latitude` | DECIMAL(10,8) | GPS latitude |
| `longitude` | DECIMAL(11,8) | GPS longitude |
| `total_surface` | DECIMAL(12,2) | Area in m² |
| `country` | VARCHAR(100) | Country |

---

## 5. API Specification

### 5.1 Update Profile

```
PATCH /api/v2/users/{userId}
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "first_name": "John",
  "last_name": "Smith",
  "email": "john@example.com",
  "whatsapp_country": "+1",
  "whatsapp_number": "5551234567"
}

Response: 200 OK
{
  "message": "Profile updated successfully"
}
```

### 5.2 Change Password

```
POST /api/v2/change-password
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "old_password": "***",
  "new_password": "***",
  "confirm_password": "***"
}

Response: 200 OK
{
  "message": "Password changed successfully"
}
```

### 5.3 Save Dashboard Settings

```
PATCH /api/v2/dashboard-settings/{userId}
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "linked_devices": {
    "et": "device-uuid",
    "rainfall": "device-uuid",
    "wind": "device-uuid"
  },
  "disease_models": ["fusarium", "anthracnose", "dollar-spot"],
  "performance_metrics": ["gdd", "et", "soil-moisture"]
}

Response: 200 OK
{
  "message": "Dashboard settings saved"
}
```

### 5.4 Create API Token

```
POST /api/v2/api-tokens
Content-Type: application/json
Authorization: Bearer {token}

Request Body:
{
  "name": "External Integration",
  "scope": ["read:weather", "read:soil"],
  "expires_in_days": 365
}

Response: 201 Created
{
  "id": "token-uuid",
  "token": "maya_sk_live_xxxxx",
  "expires_at": "2026-12-30T00:00:00Z"
}
```

### 5.5 Get Devices

```
GET /api/v2/devices?idtenant={tenant}
Authorization: Bearer {token}

Response: 200 OK
{
  "devices": [
    {
      "id": "device-uuid",
      "device_name": "SOIL_001",
      "display_name": "Green 1 Sensor",
      "device_type": "soil",
      "assigned_zone": "zone-uuid",
      "status": "active"
    }
  ]
}
```

---

## 6. Component Registration

### 6.1 setting.vue Components

```javascript
// Registered components in setting.vue
components: {
  MyAccount,
  ChangePassword,
  BussinessType,
  DashboardSetting,
  AlarmSetting,
  GetStartedSetting,
  AgronomySetting,
  VWCSeting,
  SystemSetting,
  ScheduleSetting,
  WaterSetting,
  GeneralSetting,
  FleetSetting,
  InventorySetting,
  MyConnectedDevices,
  GreassRootSetting,
  ApiTokenSetting,
  CustomToast,
  ReportSetting
}
```

### 6.2 Tab Configuration

```javascript
// Tab structure in created() hook
tab: [
  { code: 'MyAccount', label: 'my_account', icon: 'bi-person' },
  { code: 'BussinessType', label: 'my_business', icon: 'bi-building' },
  { code: 'ChangePassword', label: 'change_password', icon: 'bi-lock' },
  { code: 'GetStartedSetting', label: 'setup_my_site', icon: 'bi-geo' },
  { code: 'MyConnectedDevices', label: 'devices', icon: 'bi-cpu' },
  { code: 'AgronomySetting', label: 'agronomy', icon: 'bi-tree' },
  { code: 'DashboardSetting', label: 'dashboard', icon: 'bi-grid' },
  { code: 'VWCSeting', label: 'portable_soil', icon: 'bi-thermometer' },
  { code: 'ScheduleSetting', label: 'schedule', icon: 'bi-calendar' },
  { code: 'WaterSetting', label: 'water', icon: 'bi-droplet' },
  { code: 'InventorySetting', label: 'inventory', icon: 'bi-box' },
  { code: 'FleetSetting', label: 'fleet', icon: 'bi-truck' },
  { code: 'ReportSetting', label: 'report', icon: 'bi-file-text' },
  { code: 'ApiTokenSetting', label: 'api_token', icon: 'bi-key' },
  { code: 'GeneralSetting', label: 'general', icon: 'bi-gear' }
]
```

---

## 7. Integration Points

| System | Type | Data Flow |
|:---|:---|:---|
| All Modules | Consumer | Read configuration |
| Authentication | Provider | Password, tokens |
| Hardware | Reference | Device mapping |
| External APIs | Provider | Token access |
| Mobile App | Consumer | Notification prefs |

---

## 8. Performance Requirements

| Metric | Requirement |
|:---|:---|
| Settings page load | < 2 seconds |
| Configuration save | < 1 second |
| Device list load | < 2 seconds |
| Token generation | < 1 second |

---

**Document End**
