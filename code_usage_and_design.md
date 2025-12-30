# Technical Documentation: Tasks & Nutritional Inputs (AS-IS)

## 1. Overview

This document provides a technical breakdown of the current implementation for **Tasks** (Planned Work) and **Nutritional Inputs** (Spraying). The system currently maintains these as separate entities in both the database and the application logic.

**Clarification:**
*   **"Nitrion Input"** = **Nutritional Input** = **Spraying** (in codebase).
*   **Codebase Root:** `core-2.0` (Backend API), `web` (Frontend).

## 2. Deep Dive: Field Usage Analysis

This section maps the raw database fields to their actual usage in the application code.

### **2.1 Task Table (Tasks)**
**Table:** `task`
**Usage:** Represents generic planned work (Mowing, Aeration, Repairs, etc.).

| Field Name | Type | Code Usage & Description |
| :--- | :--- | :--- |
| `idtask` | `binary(16)` | **Primary Key**. UUID. Used everywhere to identify the task. |
| `idgroup` | `binary(16)` | **Tenant ID**. Used for multi-tenancy filtering (`ScheduleRepository::getTaskRecordsByTenantAndDate`). |
| `idspraying`| `binary(16)` | **LEGACY / UNUSED**. Found in schema but rarely used in active Task logic. `Spraying` table has its own ID. |
| `name` | `varchar` | **Display Name**. Shown in the Calendar/Scheduler UI. |
| `task_type` | `varchar` | **Categorization**. Used to distinguish task types (e.g., "Mowing", "Rolling"). |
| `method` | `varchar` | **Unique Method Key**. Often used as a unique identifier for specific task logic or API actions. |
| `status` | `tinyint` | **Workflow Status**. `1` usually means Active/Pending. Used in `TasksForm.vue` to track state. |
| `running` | `tinyint` | **Active Flag**. `1` = In Progress, `0` = Stopped/Done. Triggers inventory adjustments when toggled (see `ScheduleController::updateTask`). |
| `task_start` | `timestamp` | **Start Time**. Critical for the Calendar view (`TasksIndex.vue`). |
| `task_end` | `timestamp` | **End Time**. Defines duration. |
| `estimate_time`| `int` | **Duration (Minutes)**. Used for planning calculations. |
| `idstaff` | `binary(16)` | **Assigned Staff**. Links to `User` model. **Note:** A "Task" usually represents *one* staff member's work. Multi-staff tasks are multiple rows with same `merge_key`. |
| `idmachinery`| `binary(16)` | **Assigned Machine**. Links to `Machine` model. |
| `idsite` | `binary(16)` | **Location**. Links to `Site`. |
| `task_parameters`| `mediumtext`| **Dynamic Data (JSON)**. Stores specific attributes not in main schema. **Heavily used for Mowing** (e.g., `MowingTaskParameter`) to store patterns, directions, and sub-site details. |
| `merge_key` | `longtext` | **Grouping Key**. **Critical**. Used to link multiple `task` rows that belong to the same logical operation (e.g., same job split across 3 staff). `ScheduleController` uses this to update/delete all related tasks at once. |
| `merged_time` | `timestamp` | **Sync Time**. Likely used to track when the merge/split happened or for UI grouping. |
| `recurrence_rule`| `text` | **RRule**. Stores standard RRule string (RFC 5545) for recurring events (e.g., "FREQ=WEEKLY;BYDAY=MO"). |
| `recurrent` | `tinyint` | **Recurrence Flag**. `1` if the task repeats. |
| `notification_via`| `varchar`| **Alert Channel**. 'email', 'push', etc. |
| `version` | `int` | **Optimistic Locking**. Incremented on updates to prevent overwrites. |

### **2.2 Spraying Table (Nutritional Inputs)**
**Table:** `spraying`
**Usage:** Represents chemical/fertilizer applications. Handled by `SprayingController`.

| Field Name | Type | Code Usage & Description |
| :--- | :--- | :--- |
| `idspraying` | `binary(16)` | **Primary Key**. UUID. |
| `idproduct` | `binary(16)` | **Product Applied**. Links to `Product` model. Critical for N-P-K calculations. |
| `quantity` | `double` | **Amount Used**. fundamental input for inventory deduction and nutritional reporting. |
| `dilution_volume`| `int` | **Water Volume**. Used for tank mix calculations. |
| `quantity_area_unit`| `double`| **Rate**. Quantity per area unit (e.g., kg/ha). |
| `n_units` | `double` | **Nitrogen Units**. Calculated field or manual override for precise N tracking (`SprayingController::getNitrogenUnitsPerProductQuantity`). |
| `nozzle_type` | `int` | **Simple Nozzle**. Basic integer enum for nozzle type. |
| `idnozzle_type`| `int` | **Advanced Nozzle**. Links to `NozzleType` table (Newer compliance feature). |
| `pressure` | `double` | **Application Pressure**. Compliance/Audit data. |
| `truck_speed`| `double` | **Vehicle Speed**. Compliance/Audit data. |
| `rpm` | `double` | **Engine RPM**. Machinery tracking data. |
| `idsite` | `binary(16)` | **Target Site**. Where the chemical was applied. |
| `doneon` | `datetime` | **Completion Date**. Used for historical records and reporting. |
| `task_start` | `timestamp` | **Planned Start**. Aligning with Task model for calendar views. |
| `spraying_no`| `int` | **Sequence Number**. User-friendly ID for reference. |

---

## 3. Database Schema (Raw SQL)

Extracted from `maya-prod-dev-20251003.sql`.

### **3.1 Task Table** (`task`)

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

### **3.2 Spraying Table** (`spraying`)

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

---

## 4. API & Controller Logic

### **4.1 ScheduleController (`ScheduleController.php`)**
*   **Key Logic**:
    *   **Merge Key**: Updates matching `merge_key` rows together.
    *   **Inventory**: Deducts stock when `running` goes `0` -> `1` (via `scheduleInventoryAdjustment`).
    *   **Calendar**: Merges `task` and `spraying` rows manually in `getSchedulesForCalender` to present a unified frontend view.

### **4.2 SprayingController (`SprayingController.php`)**
*   **Key Logic**:
    *   **Calculators**: specific endpoints `getNitrogenUnitsPerProductQuantity` to help users calculate correct dosage.
    *   **Reporting**: Feeds `NutritionalReports` which aggregate `quantity` * `product.n_percentage` over time.
