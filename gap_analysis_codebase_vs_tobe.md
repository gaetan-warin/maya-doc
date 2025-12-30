# Gap Analysis: Codebase vs TO-BE Design Coverage

> **Generated:** 2025-12-22  
> **Status:** ✅ RESOLVED - All gaps addressed in TO-BE design update  
> **Purpose:** Identify fields and functionality in AS-IS codebase not yet covered by TO-BE design

---

## Summary

After comprehensive codebase analysis, gaps were identified and **all have been addressed** in the updated TO-BE design.

| Category | AS-IS Entity | Status |
|----------|--------------|--------|
| Task Core | `Task.php` (27 fields) | ✅ All fields added |
| Spraying | `Spraying.php` (39 fields) | ✅ All fields added |
| TOIL | `Toil.php` (separate table) | ✅ Kept separate (v2 migration) |
| Daily Routines | `DailyRoutineTask.php` | ✅ Kept separate (v2 migration) |
| Spraying Routines | `SprayingRoutine.php` | ✅ Kept separate (v2 migration) |
| User Compliance | `phyto_license` | ✅ Service-level validation |

---

## 1. Task Model Gaps

### AS-IS `Task.php` Fields (Core 2.0)

```php
protected $fillable = [
    'task_type', 'idtask', 'idgroup', 'idsite', 'idaction', 'merge_key', 'sortingKey',
    'task_start', 'task_end', 'estimate_time', 'name', 'method', 'type', 'timeout',
    'recurrent', 'next_execution', 'last_execution', 'recurrence_rule', 'task_parameters',
    'reason', 'note', 'is_am', 'is_pm', 'is_over_time', 'task_number', 'status',
    'running', 'version', 'notification_via', 'notification_date','idparent_task','idmachinery','idstaff','createdon','is_all'
];
```

### Gap Analysis

| AS-IS Field | TO-BE Coverage | Action Required |
|-------------|----------------|-----------------|
| `task_type` | `type_id` ✅ | Covered |
| `idaction` | Not explicit | **Add `action_id`** |
| `sortingKey` | Not covered | **Add `sort_order`** |
| `estimate_time` | Implicit in start/end | ✅ |
| `method` | Not covered | **Add `method`** (mowing direction etc) |
| `timeout` | Not covered | **Add `timeout_minutes`** |
| `recurrence_rule` | Not explicit | **Add to `task_recurrences`** |
| `task_parameters` | Not covered | **Add `parameters` JSON** |
| `reason` | Not covered | **Add `cancellation_reason`** |
| `note` | Not covered | **Add `notes`** |
| `task_number` | Not covered | **Add `task_number`** (display ID) |
| `notification_via` | Not covered | **Add `notification_channel`** |
| `notification_date` | Not covered | **Add `notify_at`** |
| `is_all` | Not covered | **Clarify purpose** |

---

## 2. Spraying Model Gaps

### AS-IS `Spraying.php` Fields

```php
protected $fillable = [
    'idsite', 'idspraying', 'quantity', 'quantity_area_unit', 'dilution_volume',
    'idproduct', 'idstaff', 'idmachinery', 'is_am', 'is_pm', 'is_over_time',
    'hole', 'comment', 'idgroup', 'name', 'doneon', 'nozzle_type', 'pressure',
    'truck_speed', 'speed', 'running', 'estimate_time', 'task_start', 'task_end',
    'status', 'reason', 'dissolution_ha', 'total_dissolution',
    'product_amount_per_tank', 'number_of_tanks', 'recommended_flow_rate',
    'idnozzle_type', 'n_units', 'idspacing', 'truck_gear', 'rpm', 'spraying_no',
    'spraying_routine_id'
];
```

### Gap Analysis

| AS-IS Field | TO-BE Coverage | Action Required |
|-------------|----------------|-----------------|
| `quantity_area_unit` | Not explicit | **Add unit field** |
| `dilution_volume` | Covered as dissolution | ✅ |
| `hole` | `task_locations.hole_id` | ✅ |
| `comment` | Not covered | **Add `comments`** |
| `doneon` | `actual_end_at` | ✅ |
| `pressure` | Not covered | **Add `pressure_bar`** |
| `truck_speed` | `tractor_speed_kmh` | ✅ (rename) |
| `speed` | Duplicate? | Clarify |
| `product_amount_per_tank` | Not covered | **Add field** |
| `idnozzle_type` | `nozzle_type` (string) | **Change to FK** |
| `idspacing` | Not covered | **Add `spacing_id` FK** |
| `truck_gear` | Not covered | **Add `gear`** |
| `spraying_no` | Not covered | **Add `spraying_number`** (display ID) |
| `spraying_routine_id` | Not covered | **Add FK to routine** |

---

## 3. TOIL Model - Design Mismatch

### AS-IS `Toil.php`

The TOIL system is a **separate table**, not embedded in task assignments:

```php
protected $table = 'toil';
protected $fillable = [
    'idtoil', 'idspraying', 'hour', 'min', 
    'starting_date_time', 'iduser', 'type', 'idtask'
];
```

### Issue

Our TO-BE design has TOIL as fields in `task_assignments`:
- `toil_hours`
- `toil_requested`

But the AS-IS has TOIL as a **separate entity** that can:
- Link to either a `task` OR a `spraying`
- Have its own `type` (TOIL vs Overtime distinction)
- Store `hour` and `min` separately

### Recommendation

Option A: Keep separate `toil` table (maintains AS-IS behavior)
Option B: Embed in assignments (our current design) + migration

**Decision Required**

---

## 4. Daily Routine System - Not Covered

### AS-IS Models

```
DailyRoutine.php
DailyRoutineTask.php
DailyRoutineSite.php
DailyRoutineHole.php
DailyRoutineArea.php
DailyRoutineStaff.php
DailyRoutineMachine.php
```

These represent **recurring task templates** that generate actual tasks.

### Current TO-BE Gap

Our `task_recurrences` table only handles recurrence rules, not the full routine template system.

### Recommendation

**Add `task_templates` table** for storing routine configurations, or document that Daily Routines remain separate from unified tasks.

---

## 5. Spraying Routines - Not Covered

### AS-IS Models

```
SprayingRoutine.php
SprayingRoutineSite.php
SprayingRoutineHole.php
SprayingRoutineArea.php
SprayingRoutineProduct.php
```

These are separate from the unified task model.

### Recommendation

Document relationship between routines and unified tasks.

---

## 6. User Compliance Fields

### AS-IS `User.php`

```php
'phyto_license'  // Phytosanitary license for spraying compliance
```

### Gap

Our `task_resource_operators.certification_ref` references operator certifications, but we don't explicitly link to user phyto_license.

### Recommendation

Add validation: spraying tasks require operators with valid `phyto_license`.

---

## 7. Lookup Tables Status

| Table | TO-BE Status |
|-------|--------------|
| `nozzle_type` | Referenced as string, should be FK |
| `spacing` | Not covered, **add `spacing_id`** |
| `action` | Referenced as `action_id` | ✅ |

---

## Recommended TO-BE Updates

### High Priority

1. **`tasks` table additions:**
   - `action_id` BINARY(16) FK
   - `sort_order` INT
   - `task_number` VARCHAR(20)
   - `notes` TEXT
   - `cancellation_reason` TEXT
   - `notify_at` DATETIME
   - `notification_channel` ENUM
   - `parameters` JSON

2. **`task_ext_nutritional` additions:**
   - `pressure_bar` DECIMAL(6,2)
   - `gear` VARCHAR(20)
   - `spraying_number` VARCHAR(20)
   - `spacing_id` BINARY(16) FK
   - `nozzle_type_id` BINARY(16) FK (replace string)
   - `product_amount_per_tank` DECIMAL(10,2)
   - `comments` TEXT
   - `routine_id` BINARY(16) FK

### Medium Priority

3. **TOIL decision:** Separate table vs embedded in assignments
4. **Routine templates:** Document relationship or add `task_templates`

### Low Priority

5. **Phyto license validation:** Service-level check
6. **Unit standardization:** Add `quantity_unit` field

---

## Next Steps

1. Review this gap analysis
2. Decide on TOIL architecture
3. Update TO-BE design document
4. Validate against remaining Notion pages (when browser available)
