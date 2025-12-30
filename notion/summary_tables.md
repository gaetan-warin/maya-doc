# Summary Tables Database Structure and Data Insertion Process

## Overview
The summary tables system is designed to aggregate and store processed data from various operational activities including spraying, scheduling, leave management, and overtime tracking. This document outlines the complete database structure and the data insertion process flow.

## Database Structure

### 1. Main Summary Table: `summary_tasks`
The central table that stores aggregated task information across different record types.

**Table Structure**
> Note: This table serves as the primary aggregation point for all operational activities.

```sql
CREATE TABLE summary_tasks (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    
    -- Task-specific identifiers
    idtask CHAR(36) NULL,
    spraying_session_token CHAR(64) NULL UNIQUE,
    idaction CHAR(36) NULL,
    idleave_type CHAR(36) NULL,
    idleave CHAR(36) NULL,
    idtoil CHAR(36) NULL,
    
    -- Site and tenant information
    idtenant CHAR(36) NULL,
    idgroup CHAR(36) NULL,
    
    -- Task details
    action_name VARCHAR(255) NULL,
    leave_type_name VARCHAR(255) NULL,
    site_name VARCHAR(255) NULL,
    
    -- Date and time information
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    completed_date DATE NOT NULL,
    toil_at TIMESTAMP NULL,
    
    -- Time tracking
    weeknumber INT NOT NULL,
    year INT NOT NULL,
    month INT NOT NULL,
    work_hours DECIMAL(6,2) NULL,
    estimated_time DECIMAL(6,2) NULL,
    
    -- Cost information
    total_resouce_cost DECIMAL(6,2) NULL,
    
    -- Record classification
    record_type ENUM('task_schedule','task_spraying','leave', 'toil', 'overtime') NOT NULL,
    bussiness_type ENUM('golf','football_pitch','grass_root') NOT NULL,
    
    -- Status fields
    schedule_status TINYINT NULL DEFAULT 0,  -- Approved - 1, Not Approved - 0
    leave_status TINYINT NULL DEFAULT 0,     -- Approved - 1, Not Approved - 0
    
    -- Additional fields
    merge_key LONGTEXT NULL,
    schedule_task_type ENUM('tenant-based', 'site-based') NULL,
    active_substance JSON NULL,
    product_substance JSON NULL,
    
    -- Timestamps
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    
    -- Indexes
    INDEX idx_summery_record_type (record_type),
    INDEX idx_summery_date (completed_date),
    INDEX idx_summery_tenant_group (idtenant, idgroup),
    INDEX idx_summery_business_type (bussiness_type),
    INDEX idx_summery_week (weeknumber)
);
```

**Key Fields Explanation**

| Field | Type | Description | Used For |
| :--- | :--- | :--- | :--- |
| `record_type` | ENUM | Defines the type of operation | All records |
| `spraying_session_token` | CHAR(64) | Unique identifier for spraying sessions | Spraying only |
| `idtask` | CHAR(36) | Task identifier | Schedule only |
| `idleave_type` | CHAR(36) | Leave type identifier | Leave only |
| `work_hours` | DECIMAL(6,2) | Total work hours | All records |
| `total_resouce_cost` | DECIMAL(6,2) | Resource cost calculation | Spraying only |

### 2. Related Summary Tables

#### 2.1 `summary_site` - Site Information
```sql
CREATE TABLE summary_site (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    summary_id BIGINT UNSIGNED NOT NULL,
    idsite CHAR(36) NOT NULL,
    site_name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (summary_id) REFERENCES summary_tasks(id) ON DELETE CASCADE,
    INDEX idx_summary_site (summary_id, idsite)
);
```

#### 2.2 `summary_task_staff` - Staff Information
```sql
CREATE TABLE summary_task_staff (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    summary_id BIGINT UNSIGNED NOT NULL,
    iduser CHAR(36) NULL,
    first_name VARCHAR(255) NULL,
    last_name VARCHAR(255) NULL,
    total_labour_cost DECIMAL(6,2) NULL,
    work_hours DECIMAL(6,2) NULL,
    overtime DECIMAL(6,2) NULL,
    leave_days DECIMAL(6,2) NULL,
    toil DECIMAL(6,2) NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (summary_id) REFERENCES summary_tasks(id) ON DELETE CASCADE
);
```

#### 2.3 `summary_task_tag` - Tag Information
Associates tags and categories with summary records.

#### 2.4 `summary_task_subsite` - Subsite Information
Links subsites (holes/areas) to summary records.

#### 2.5 `summary_task_machinery` - Machinery Information
Tracks machinery used in tasks.

#### 2.6 `summary_task_spraying_sessions` - Spraying Session Tracking

#### 2.7 `summary_task_spraying_products` - Product Usage Tracking

## Data Insertion Process Flow

### 1. Event-Driven Architecture
`User Action` → `Repository Method` → `Event Dispatch` → `Listener` → `Service Processing` → `Database Insertion`

### 2. Record Types and Processing
*   **2.1 Spraying Records (`task_spraying`)**: Triggered by `createSprayingRecords()`, uses `UpdateSprayingSummery` listener and `processSprayingSummery` service.
*   **2.2 Schedule Records (`task_schedule`)**: Triggered by `createScheduleRecords()`, uses `UpdateScheduleSummary` listener and `processScheduleSummery` service.
*   **2.3 Leave Records (`leave`, `toil`, `overtime`)**: Uses `UpdateLeaveSummary` listener and `processLeaveSummary` service.

### 3. Data Processing Steps
*   **3.1 Data Formatting**: Services format raw data into a standardized structure.
*   **3.2 Database Transaction Handling**: All summary data insertions are wrapped in database transactions.

(Archived from Notion on 2025-12-22)
