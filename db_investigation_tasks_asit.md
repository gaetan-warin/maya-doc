# DB Investigation & Technical Documentation: Tasks (AS-IS)

## 1. Executive Summary

This document details the **current ("AS-IS") technical implementation** of the Task Management and Nutritional Input systems in the `mayaApp` codebase.

**Core Findings:**
*   **Separation**: Tasks (`task`) and Chemical Applications (`spraying`) are strictly separate tables.
*   **Access Pattern**: The application primarily uses **Eloquent ORM** via the core-2.0 API, but retains **Legacy OData Calls** for specific modules (Fleet, Product Categories).
*   **Architecture**:
    *   **Backend**: `core-2.0` (Laravel) serves as the main API.
    *   **Frontend**: `web` (Vue.js) uses Vuex Stores.
    *   **Legacy**: `core` (Core 1) is still accessed directly by the Frontend for specific OData queries via a generic wrapper.

---

## 2. Field Usage Analysis (Deep Dive)

### **2.1 Task Table** (`task`)

| Field | Type | Code Usage & Investigation Findings |
| :--- | :--- | :--- |
| **`idtask`** | `binary(16)` | **Primary Key**. UUID. |
| **`task_number`** | `int` | **Reference ID**. Readable ID. |
| **`idgroup`** | `binary(16)` | **Tenant Context**. |
| **`idspraying`** | `binary(16)` | **Legacy/Unused**. |
| **`idrule`** | `binary(16)` | **Automation**. |
| **`name`** | `varchar` | **UI Label**. |
| **`method`** | `varchar` | **Unique Method Key**. |
| **`type`** | `int` | **Discriminator**. 0=Schedule task. |
| **`timeout`** | `int` | **Execution Limit**. |
| **`recurrent`** | `tinyint` | **Recurrence Flag**. |
| **`next_execution`**| `datetime` | **Recurrence Logic**. |
| **`last_execution`**| `datetime` | **Recurrence Logic**. |
| **`recurrence_rule`**| `text` | **RFC 5545 RRule**. |
| **`task_parameters`**| `mediumtext`| **Dynamic Payload**. JSON (Mowing patterns, etc). |
| **`status`** | `tinyint` | **Workflow State**. |
| **`running`** | `tinyint` | **Active Flag**. |
| **`version`** | `int` | **Optimistic Locking**. |
| **`createdon`** | `timestamp` | **Audit**. |
| **`modifiedon`** | `timestamp` | **Audit**. |
| **`idstaff`** | `binary(16)` | **Assigned Worker**. |
| **`idmachinery`**| `binary(16)` | **Equipment**. |
| **`idaction`** | `binary(16)` | **Action Template**. |
| **`iduser`** | `binary(16)` | **Creator/Owner**. |
| **`reason`** | `varchar` | **Audit/Log**. |
| **`idsite`** | `binary(16)` | **Location**. |
| **`note`** | `varchar` | **User Notes**. |
| **`task_start`** | `timestamp` | **Scheduling**. |
| **`task_end`** | `timestamp` | **Scheduling**. |
| **`idparent_task`**| `binary(16)` | **Hierarchy**. |
| **`estimate_time`**| `int` | **Planning**. |
| **`notification_via`**| `varchar` | **Alerts**. |
| **`notification_date`**| `timestamp`| **Alerts**. |
| **`is_am`** | `tinyint` | **Scheduling**. |
| **`is_pm`** | `tinyint` | **Scheduling**. |
| **`is_over_time`** | `tinyint` | **Payroll**. |
| **`is_all`** | `tinyint` | **Scope**. |
| **`merge_key`** | `longtext` | **Grouping**. |
| **`merged_time`** | `timestamp` | **Grouping**. |
| **`task_type`** | `varchar` | **Categorization**. |

### **2.2 Spraying Table** (`spraying`)

| Field | Type | Code Usage & Investigation Findings |
| :--- | :--- | :--- |
| **`idspraying`** | `binary(16)` | **Primary Key**. UUID. |
| **`idgroup`** | `binary(16)` | **Tenant Context**. |
| **`idsite`** | `binary(16)` | **Target Site**. |
| **`idproduct`** | `binary(16)` | **Chemical**. |
| **`name`** | `varchar` | **Label**. |
| **`doneon`** | `datetime` | **Completion Date**. |
| **`quantity`** | `double` | **Total Amount**. |
| **`dilution_volume`**| `int` | **Water Mix**. |
| **`version`** | `int` | **Optimistic Locking**. |
| **`createdon`** | `timestamp` | **Audit**. |
| **`modifiedon`** | `timestamp` | **Audit**. |
| **`quantity_area_unit`**| `double`| **Rate**. |
| **`is_am`** | `tinyint` | **Scheduling**. |
| **`is_pm`** | `tinyint` | **Scheduling**. |
| **`is_over_time`** | `tinyint` | **Payroll**. |
| **`idmachinery`** | `binary(16)` | **Equipment**. |
| **`idstaff`** | `binary(16)` | **Operator**. |
| **`running`** | `tinyint` | **Active Flag**. |
| **`estimate_time`**| `int` | **Planning**. |
| **`task_start`** | `timestamp` | **Planned Start**. |
| **`task_end`** | `timestamp` | **Planned End**. |
| **`reason`** | `varchar` | **Audit**. |
| **`speed`** | `double` | **Compliance**. |
| **`nozzle_type`** | `int` | **Hardware**. |
| **`pressure`** | `double` | **Compliance**. |
| **`truck_speed`** | `double` | **Compliance**. |
| **`truck_gear`** | `varchar` | **Compliance**. |
| **`rpm`** | `double` | **Compliance**. |
| **`comment`** | `varchar` | **Notes**. |
| **`hole`** | `varchar` | **Location**. |
| **`status`** | `tinyint` | **State**. |
| **`idnozzle_type`**| `int` | **Hardware**. |
| **`dissolution_ha`**| `double` | **Calculation**. |
| **`total_dissolution`**| `double`| **Calculation**. |
| **`product_amount_per_tank`**|`double`| **Calibration**. |
| **`number_of_tanks`**| `double` | **Calibration**. |
| **`recommended_flow_rate`**|`double`| **Calibration**. |
| **`n_units`** | `double` | **Nitrogen**. |
| **`idspacing`** | `int` | **Configuration**. |
| **`spraying_no`** | `int` | **Reference ID**. |

---

## 3. Data Access Layer & API Map

### **3.1 Access Method Analysis**
*   **Eloquent ORM**: 95% of operations.
*   **Raw SQL**: Reporting/Calculations only.

### **3.2 API Interaction Map (Core 2.0)**

#### **A. Tasks (`task` table)**
| Endpoint (v2) | Method | Controller | Access Type |
| :--- | :--- | :--- | :--- |
| `/schedules` | `GET` | `ScheduleController@getSchedulesForCalender` | Eloquent |
| `/createSchedule` | `POST` | `ScheduleController@createSchedule` | Eloquent |
| `/update-schedule` | `PUT` | `ScheduleController@updateTask` | Eloquent |
| `/schedule/{mergeKey}`| `DELETE`| `ScheduleController@deleteSchedule` | Eloquent |

#### **B. Spraying (`spraying` table)**
| Endpoint (v2) | Method | Controller | Access Type |
| :--- | :--- | :--- | :--- |
| `/createSprayingRecords`| `POST` | `SprayingController@createSprayingRecords` | Eloquent |
| `/getSprayingRecords` | `GET` | `SprayingController@getSprayingRecords` | Eloquent |

---

## 4. Full Architecture Usage Map

### **4.1 Frontend (`web` - Vue.js)**

The frontend does NOT access the DB directly. It consumes the Core 2.0 API via Vuex Actions.

| Module | Store/Code File | Usage Description |
| :--- | :--- | :--- |
| **Scheduling** | `store/schedule/index.js` | **Main Store**. Actions: `createSchedule`, `updateSchedule`, `fetchSchedules`. |
| **Spraying** | `store/spraying/index.js` | **Main Store**. Actions: `createSprayingRecord`, `fetchSprayingRecords`. |
| **Component** | `views/setting.vue` | General configuration screens that might touch task settings. |
| **Component** | `components/schedule/scheduleTasks/ScheduleIndex.vue` | **Main Calendar**. Calls `getSchedulesForCalender`. |
| **Component** | `components/schedule-v2/edit-task/form.vue` | **Edit UI**. Calls `updateTask`. |

### **4.2 Legacy Core (`core` - PHP/OData)**

> [!WARNING]
> **Active Legacy Usage Detected**: The frontend `web` **directly calls Core 1** for specific OData queries using `{ useLegacyApi: true }` in `store/axios-config.js`.

*   **Mechanism**: `axios-config.js` swaps `baseURL` to `VITE_API_URL` (Core 1) when the flag is present.
*   **Verified Usage Sites**:
    1.  **Spraying Store** (`store/spraying/index.js`):
        *   `getProductCategories()` -> `/product_category?$expand=...`
        *   `retrievePressure()` -> `/unit?$filter=...`
    2.  **Fleet Store** (`store/fleet.js`):
        *   `getMaintenanceDisplay()` -> `/machine-maintenance-by-tenant/...`
        *   `getMachineData()` -> `/getMachineData(...)`
*   **Implication**: While `Tasks` are fully migrated to Core 2.0, **Fleet** and some **Lookup Data (Product Categories/Units)** still rely on the Legacy Core OData interface.

---

## 5. Notion & Architecture Findings (Contextual Analysis)

Based on a review of the internal Notion documentation ("Task, Labour & Nutritional Input"), the following context explains the AS-IS state:

### **5.1 Identified Pain Points ("AS-IS")**
*   **Data Fragmentation**: The strict separation of `task` and `spraying` tables is identified as a "historical artefact" causing significant technical debt.
*   **API Inefficiency**: The current Schedule module is documented to make **20+ redundant API calls per page load** due to this fragmentation.
*   **Reporting Bugs**: Known issues exist with mismatched units (`kg/ha` vs `g/sqm`) and empty NPK values in legacy reports.

### **5.2 Key Term Definitions**
*   **`merge_key`**: A critical identifier used to group multiple database rows (e.g., one task split across 3 staff members) into a single logical "Task" for the user. Updates to a `merge_key` propagate to all child rows.
*   **`task_parameters`**: Validated as a **JSON container** for dynamic task data (e.g., mowing patterns, directions) that doesn't fit standard columns.
*   **`summary_tasks` (TO-BE Model)**: The documentation references a future "Summary Table" architecture intended to unify `tasks`, `spraying`, `leaves`, and `overtime` into a single high-performance table. This confirms why the current `task` table seems to have unused foreign keys (like `idspraying`)—they are vestiges of partial or future integration attempts.

---

## 6. Raw Database Schema (`SHOW CREATE TABLE`)

### **6.1 Task**
```sql
CREATE TABLE `task` (
  `idtask` binary(16) NOT NULL,
  `task_number` int DEFAULT NULL,
  `idgroup` binary(16) DEFAULT NULL,
  `idspraying` binary(16) DEFAULT NULL,
  `idrule` binary(16) DEFAULT NULL,
  `name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  `method` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
  `type` int NOT NULL DEFAULT '0' COMMENT 'Value type 0 = Schedule task and 1 = Spraying notification',
  `timeout` int NOT NULL DEFAULT '0',
  `recurrent` tinyint unsigned NOT NULL DEFAULT '0',
  `next_execution` datetime DEFAULT NULL,
  `last_execution` datetime DEFAULT NULL,
  `recurrence_rule` text CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci,
  `task_parameters` mediumtext CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci,
  `status` tinyint NOT NULL DEFAULT '1',
  `running` tinyint NOT NULL DEFAULT '0',
  `version` int NOT NULL DEFAULT '0',
  `createdon` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `modifiedon` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `idstaff` binary(16) DEFAULT NULL,
  `idmachinery` binary(16) DEFAULT NULL,
  `idaction` binary(16) DEFAULT NULL,
  `iduser` binary(16) DEFAULT NULL,
  `reason` varchar(120) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  `idsite` binary(16) DEFAULT NULL,
  `note` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  `task_start` timestamp NULL DEFAULT NULL,
  `task_end` timestamp NULL DEFAULT NULL,
  `idparent_task` binary(16) DEFAULT NULL,
  `estimate_time` int unsigned DEFAULT NULL,
  `notification_via` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT 'email',
  `notification_date` timestamp NULL DEFAULT NULL,
  `is_am` tinyint DEFAULT NULL,
  `is_pm` tinyint DEFAULT NULL,
  `is_over_time` tinyint(1) DEFAULT '0',
  `is_all` tinyint(1) NOT NULL DEFAULT '0',
  `merge_key` longtext CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci,
  `merged_time` timestamp NULL DEFAULT NULL,
  `task_type` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  PRIMARY KEY (`idtask`),
  UNIQUE KEY `task_method_unique` (`method`),
  KEY `subtask task` (`idparent_task`),
  CONSTRAINT `subtask task` FOREIGN KEY (`idparent_task`) REFERENCES `task` (`idtask`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### **6.2 Spraying**
```sql
CREATE TABLE `spraying` (
  `idspraying` binary(16) NOT NULL,
  `idgroup` binary(16) NOT NULL,
  `idsite` binary(16) NOT NULL,
  `idproduct` binary(16) NOT NULL,
  `name` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  `doneon` datetime DEFAULT NULL,
  `quantity` double DEFAULT NULL,
  `dilution_volume` int unsigned DEFAULT NULL,
  `version` int NOT NULL DEFAULT '0',
  `createdon` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `modifiedon` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `quantity_area_unit` double(16,5) DEFAULT NULL,
  `is_am` tinyint DEFAULT NULL,
  `is_pm` tinyint DEFAULT NULL,
  `is_over_time` tinyint DEFAULT '0',
  `idmachinery` binary(16) DEFAULT NULL,
  `idstaff` binary(16) DEFAULT NULL,
  `running` tinyint NOT NULL DEFAULT '0',
  `estimate_time` int unsigned DEFAULT NULL,
  `task_start` timestamp NULL DEFAULT NULL,
  `task_end` timestamp NULL DEFAULT NULL,
  `reason` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  `speed` double(8,2) DEFAULT NULL,
  `nozzle_type` int DEFAULT NULL,
  `pressure` double(8,2) DEFAULT NULL,
  `truck_speed` double(8,2) DEFAULT NULL,
  `truck_gear` varchar(255) COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  `rpm` double DEFAULT NULL,
  `comment` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  `hole` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  `status` tinyint DEFAULT '1',
  `idnozzle_type` int DEFAULT NULL,
  `dissolution_ha` double DEFAULT NULL,
  `total_dissolution` double DEFAULT NULL,
  `product_amount_per_tank` double DEFAULT NULL,
  `number_of_tanks` double DEFAULT NULL,
  `recommended_flow_rate` double DEFAULT NULL,
  `n_units` double DEFAULT NULL,
  `idspacing` int DEFAULT NULL,
  `spraying_no` int DEFAULT NULL,
  PRIMARY KEY (`idspraying`),
  KEY `spraying group` (`idgroup`),
  KEY `spraying site` (`idsite`),
  KEY `spraying product` (`idproduct`),
  CONSTRAINT `spraying group` FOREIGN KEY (`idgroup`) REFERENCES `group` (`idgroup`) ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT `spraying product` FOREIGN KEY (`idproduct`) REFERENCES `product` (`idproduct`) ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT `spraying site` FOREIGN KEY (`idsite`) REFERENCES `site` (`idsite`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```
