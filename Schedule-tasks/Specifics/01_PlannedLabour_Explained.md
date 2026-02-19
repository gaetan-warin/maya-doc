# Planned Labour: How It Works Today vs. After Implementation
## Deep Dive Explanation

---

## 1. The Core Concept: What is "Planned Labour"?

**Planned Labour** is the total amount of work effort (in hours) that a task is expected to consume.

The key insight is that **task duration** and **labour** are **different things**:

| Concept | Definition | Example |
|:---|:---|:---|
| **Duration** | How long the task takes (time window) | 3 hours (8am → 11am) |
| **Labour** | Total staff-hours consumed | 6 labour hours (3h × 2 staff) |

---

## 1.5 Two Types of Duration: Planned vs. Default

There are **TWO duration concepts** in the system:

| Concept | Source | Description |
|:---|:---|:---|
| **Planned Duration** | `task.estimate_time` | User-entered duration when creating a task |
| **Default Duration** | `taggable.duration` | Pre-configured based on Action + Location |

### Default Duration is calculated based on Attribution:

```
┌─────────────────────────────────────────────────────────────────┐
│                          GOLF                                   │
│                                                                 │
│   Default Duration = Action + Work Zone (Tag)                   │
│                                                                 │
│   Table: taggable                                               │
│   Fields: idaction + idtag → duration                           │
│                                                                 │
│   Example:                                                      │
│   Action: "Mowing" + Work Zone: "Greens" = 45 min default       │
│   Action: "Mowing" + Work Zone: "Fairways" = 90 min default     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                         STADIUM                                 │
│                                                                 │
│   Default Duration = Action + Site + Holes/Pitches              │
│                                                                 │
│   Table: taggable                                               │
│   Fields: idaction + idsite + idpitch (JSON) → duration         │
│                                                                 │
│   Example:                                                      │
│   Action: "Grooming" + Site: "Stadium A" + Pitches: ["Main"]    │
│     = 60 min default                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### The `taggable` Table Structure (AS-IS)

```sql
CREATE TABLE taggable (
    idtaggable      BINARY(16) PRIMARY KEY,
    idtag           BINARY(16),      -- FK to tag (work zone for GOLF)
    idsite          BINARY(16),      -- FK to site
    idpitch         JSON,            -- Array of pitch IDs (for STADIUM)
    idaction        BINARY(16),      -- FK to action
    duration        INT,             -- DEFAULT duration in minutes
    surface         DECIMAL,         -- Surface area (for reporting)
    ...
);
```

### How Default Duration is Used:

1. **When Creating a Task**: Frontend calls `getDuration()` with selected Action + Areas/Sites
2. **System Looks Up**: Queries `taggable` table for matching `idaction` + `idtag` (GOLF) or `idaction` + `idpitch` (STADIUM)
3. **SUM of Durations**: Adds up all matching `taggable.duration` values
4. **Pre-fills Form**: The calculated default is shown to the user
5. **User Can Override**: User can accept or change the duration → stored as `estimate_time`

### Code Example (Frontend):

```javascript
// GOLF: findActionAreaDuration
const findActionAreaDuration = (action, areas) => {
  const areasList = areas.map(area => area.idtag);
  const findAction = schedule.actions?.find(a => a.idaction === action);
  const filteredTaggable = findAction?.taggable?.filter(t => areasList.includes(t.idtag));
  // SUM all matching durations
  const duration = filteredTaggable?.reduce((total, a) => total + (a.duration || 0), 0);
  return duration;
}

// STADIUM: findDurationUsingHoles
const findDurationUsingHoles = (actionId, sites) => {
  const match = schedule.actions.find(o => o.idaction === actionId);
  const items = match?.taggable?.filter(o =>
    Array.isArray(o.idpitch) && o.idpitch.every(pitchId => sites.includes(pitchId))
  );
  const duration = items?.reduce((acc, o) => acc + (o.duration || 0), 0);
  return duration;
}
```

---

## 2. How It Works TODAY (AS-IS)

### 2.1 Database Structure

```
┌───────────────────────────────────────────────────────────────────┐
│                        task TABLE (AS-IS)                         │
├───────────────────────────────────────────────────────────────────┤
│  One "logical task" = MULTIPLE database rows                      │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ PARENT ROW (idparent_task = NULL)                           │  │
│  │  - idtask: uuid-parent                                      │  │
│  │  - idaction: "Mowing"                                       │  │
│  │  - idsite: "North Course"                                   │  │
│  │  - estimate_time: 180 (minutes) ← DURATION STORED HERE      │  │
│  │  - task_start: NULL (or actual start time)                  │  │
│  │  - task_end: NULL (or actual end time)                      │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              │                                    │
│              ┌───────────────┼───────────────┐                    │
│              ▼               ▼               ▼                    │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐      │
│  │ CHILD ROW #1    │ │ CHILD ROW #2    │ │ CHILD ROW #3    │      │
│  │ idparent: uuid  │ │ idparent: uuid  │ │ idparent: uuid  │      │
│  │ idstaff: John   │ │ idstaff: Marie  │ │ idmachinery: X  │      │
│  │ estimate_time:  │ │ estimate_time:  │ │ (machine row)   │      │
│  │ NULL or copied  │ │ NULL or copied  │ │                 │      │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘      │
│                                                                   │
│  ⚠️ Staff do NOT have individual time tracking!                   │
└───────────────────────────────────────────────────────────────────┘
```

### 2.2 Key Fields (AS-IS)

| Field | Type | Location | Purpose |
|:---|:---|:---|:---|
| `estimate_time` | INT | Parent row | Planned duration in **minutes** |
| `task_start` | TIMESTAMP | Parent row | When task actually started |
| `task_end` | TIMESTAMP | Parent row | When task actually ended |
| `idstaff` | BINARY(16) | Child rows | Staff member assigned |

### 2.3 How Labour is Calculated TODAY

**There is NO explicit labour tracking.** You must calculate it manually:

```
Planned Labour = estimate_time × COUNT(staff child rows)
```

**Example:**
- Task: Mowing
- `estimate_time`: 180 minutes (3 hours)
- Child rows with `idstaff`: 2 (John, Marie)

```
Planned Labour = 180 × 2 = 360 minutes = 6 labour hours
```

### 2.4 The Problem with Today's Model

| Issue | Impact |
|:---|:---|
| **No per-staff time** | Cannot track if John worked 3h but Marie only worked 2h |
| **Same duration for all** | All staff "inherit" the parent's estimate_time |
| **Actual time = task level** | Cannot track actual time per staff member |
| **Row duplication** | One task with 5 staff = 6 database rows (1 parent + 5 children) |

### 2.5 Code Example (AS-IS)

From [StaffService.php:126-127](core-2.0/app/Services/StaffService.php#L126-L127):
```php
WHEN t.estimate_time IS NOT NULL AND t.estimate_time != 0 THEN
    SEC_TO_TIME(t.estimate_time * 60)
```

The system calculates duration from the parent row's `estimate_time` for ALL staff.

---

## 3. How It Will Work AFTER (TO-BE)

### 3.1 Database Structure

```
┌───────────────────────────────────────────────────────────────────┐
│                        tasks TABLE (TO-BE)                        │
├───────────────────────────────────────────────────────────────────┤
│  One "logical task" = ONE database row                            │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ SINGLE TASK ROW                                             │  │
│  │  - id: uuid                                                 │  │
│  │  - action_id: "Mowing"                                      │  │
│  │  - planned_minutes: 180 ← TASK DURATION                     │  │
│  │  - planned_start: 2025-01-15 08:00:00                       │  │
│  │  - planned_end: 2025-01-15 11:00:00                         │  │
│  │  - actual_start: (when started)                             │  │
│  │  - actual_end: (when completed)                             │  │
│  └─────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│                    task_assignments TABLE                         │
├───────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐ ┌─────────────────┐                          │
│  │ Assignment #1   │ │ Assignment #2   │                          │
│  │ task_id: uuid   │ │ task_id: uuid   │                          │
│  │ user_id: John   │ │ user_id: Marie  │                          │
│  │ role: LEAD      │ │ role: WORKER    │                          │
│  │ is_overtime: N  │ │ is_overtime: N  │                          │
│  │ planned_minutes:│ │ planned_minutes:│  ← PER-STAFF PLANNED     │
│  │   180 (or NULL) │ │   180 (or NULL) │                          │
│  │ actual_minutes: │ │ actual_minutes: │  ← PER-STAFF ACTUAL      │
│  │   175           │ │   190           │                          │
│  └─────────────────┘ └─────────────────┘                          │
│                                                                   │
│  ✅ Each staff member has INDIVIDUAL time tracking!               │
└───────────────────────────────────────────────────────────────────┘
```

### 3.2 Key Fields (TO-BE)

**tasks table:**
| Field | Type | Purpose |
|:---|:---|:---|
| `planned_minutes` | INT | Task duration (replaces `estimate_time`) |
| `planned_start` | DATETIME | When task should start |
| `planned_end` | DATETIME | When task should end |
| `actual_start` | DATETIME | When task actually started |
| `actual_end` | DATETIME | When task actually ended |

**task_assignments table:**
| Field | Type | Purpose |
|:---|:---|:---|
| `task_id` | BINARY(16) | FK to tasks |
| `user_id` | BINARY(16) | FK to staff member |
| `role` | VARCHAR | LEAD, WORKER, OPERATOR, SUPPORT |
| `is_overtime` | BOOLEAN | Was this overtime work? |
| `planned_minutes` | INT | **NEW:** Per-staff planned duration |
| `actual_minutes` | INT | **NEW:** Per-staff actual duration |

### 3.3 How Labour is Calculated (TO-BE)

**Option A: Simple (all staff same duration)**
```
Planned Labour = tasks.planned_minutes × COUNT(task_assignments)
```

**Option B: Explicit per-staff tracking**
```
Planned Labour = SUM(task_assignments.planned_minutes)
Actual Labour = SUM(task_assignments.actual_minutes)
```

---

## 4. The Logic Explained: "Duration × Staff"

### 4.1 The Formula

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Planned Labour Hours = Task Duration × Number of Staff        │
│                                                                 │
│   OR more precisely:                                            │
│                                                                 │
│   Planned Labour Hours = SUM(each staff's planned_minutes)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Visual Example: GOLF

```
┌─────────────────────────────────────────────────────────────────┐
│  TASK: Aeration of Greens                                       │
│  TIME: 08:00 → 11:00 (3 hours)                                  │
│  STAFF: John (Lead) + Marie (Worker)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Timeline:                                                      │
│                                                                 │
│  08:00                                               11:00      │
│    │                                                   │        │
│    ├───────────── 3 hours duration ────────────────────┤        │
│    │                                                   │        │
│    │  ┌─────────────────────────────────────────────┐  │        │
│    │  │ John works 3 hours                          │  │        │
│    │  └─────────────────────────────────────────────┘  │        │
│    │                                                   │        │
│    │  ┌─────────────────────────────────────────────┐  │        │
│    │  │ Marie works 3 hours                         │  │        │
│    │  └─────────────────────────────────────────────┘  │        │
│                                                                 │
│  RESULT:                                                        │
│  • Task Duration: 3 hours                                       │
│  • Planned Labour: 3h × 2 staff = 6 LABOUR HOURS                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 Visual Example: STADIUM

```
┌─────────────────────────────────────────────────────────────────┐
│  TASK: Pitch Grooming - Main Pitch + Training Pitch             │
│  TIME: 07:00 → 09:00 (2 hours)                                  │
│  STAFF: 4 groundskeepers                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Timeline:                                                      │
│                                                                 │
│  07:00                                     09:00                │
│    │                                         │                  │
│    ├───────── 2 hours duration ──────────────┤                  │
│    │                                         │                  │
│    │  ┌───────────────────────────────────┐  │                  │
│    │  │ Staff 1 works 2 hours             │  │                  │
│    │  │ Staff 2 works 2 hours             │  │                  │
│    │  │ Staff 3 works 2 hours             │  │                  │
│    │  │ Staff 4 works 2 hours             │  │                  │
│    │  └───────────────────────────────────┘  │                  │
│                                                                 │
│  RESULT:                                                        │
│  • Task Duration: 2 hours                                       │
│  • Planned Labour: 2h × 4 staff = 8 LABOUR HOURS                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. What Changes?

### 5.1 Summary of Changes

| Aspect | AS-IS (Today) | TO-BE (After) |
|:---|:---|:---|
| **Task rows** | Multiple rows (parent + children) | Single row |
| **Duration field** | `estimate_time` on parent | `planned_minutes` on task |
| **Staff tracking** | Child rows with `idstaff` | `task_assignments` junction table |
| **Per-staff planned time** | NOT POSSIBLE | `task_assignments.planned_minutes` |
| **Per-staff actual time** | NOT POSSIBLE | `task_assignments.actual_minutes` |
| **Labour calculation** | Manual: duration × staff count | Explicit: SUM(planned_minutes) |
| **Overtime tracking** | `is_over_time` flag on child rows | `task_assignments.is_overtime` per staff |

### 5.2 Benefits of TO-BE

| Benefit | Explanation |
|:---|:---|
| **Per-staff accuracy** | Know exactly how long each person worked |
| **Better reporting** | "John worked 45 hours this week, Marie worked 38" |
| **Overtime precision** | Track overtime per individual, not per task |
| **Flexible duration** | Lead can have 3h planned, helper can have 2h planned |
| **Simpler queries** | No need to count child rows to calculate labour |
| **Audit trail** | `actual_minutes` vs `planned_minutes` shows variance |

### 5.3 Migration Mapping

| AS-IS Field | TO-BE Field | Notes |
|:---|:---|:---|
| `task.estimate_time` | `tasks.planned_minutes` | Rename only |
| `task.task_start` | `tasks.actual_start` | Same meaning |
| `task.task_end` | `tasks.actual_end` | Same meaning |
| Child row count | `task_assignments` count | Same logic |
| `is_over_time` (child) | `task_assignments.is_overtime` | Per-staff |
| N/A | `task_assignments.planned_minutes` | **NEW** |
| N/A | `task_assignments.actual_minutes` | **NEW** |

---

## 6. Practical Scenarios

### 6.1 Scenario: Reporting Time by Site + Zone (GOLF)

**Question:** How much labour was planned/actual for "Greens" on "North Course" in January?

**AS-IS Query (complex):**
```sql
SELECT
    s.name AS site,
    tag.label AS zone,
    SUM(t.estimate_time) *
        (SELECT COUNT(*) FROM task c WHERE c.idparent_task = t.idtask AND c.idstaff IS NOT NULL)
        AS planned_labour_minutes
FROM task t
JOIN site s ON t.idsite = s.idsite
JOIN taggable tb ON t.idtask = tb.taggable_id
JOIN tag ON tb.tag_id = tag.idtag
WHERE t.idparent_task IS NULL
GROUP BY s.idsite, tag.idtag;
```

**TO-BE Query (simple):**
```sql
SELECT
    s.name AS site,
    tag.label AS zone,
    SUM(COALESCE(ta.planned_minutes, t.planned_minutes)) AS planned_labour_minutes,
    SUM(ta.actual_minutes) AS actual_labour_minutes
FROM tasks t
JOIN task_locations tl ON t.id = tl.task_id
JOIN site s ON tl.site_id = s.idsite
JOIN task_tags tt ON t.id = tt.task_id
JOIN tag ON tt.tag_id = tag.idtag
JOIN task_assignments ta ON t.id = ta.task_id
GROUP BY s.idsite, tag.idtag;
```

### 6.2 Scenario: Staff Timesheet

**Question:** How many hours did John work this week?

**TO-BE Query:**
```sql
SELECT
    u.first_name,
    u.last_name,
    SUM(ta.actual_minutes) / 60 AS actual_hours,
    SUM(ta.planned_minutes) / 60 AS planned_hours,
    (SUM(ta.actual_minutes) - SUM(ta.planned_minutes)) / 60 AS variance_hours
FROM task_assignments ta
JOIN tasks t ON ta.task_id = t.id
JOIN user u ON ta.user_id = u.iduser
WHERE u.iduser = :john_id
  AND t.planned_start BETWEEN '2025-01-06' AND '2025-01-12'
GROUP BY u.iduser;
```

---

## 7. Key Takeaways

1. **Duration ≠ Labour**
   - Duration = time window (3 hours)
   - Labour = staff-hours (3h × 2 staff = 6 labour hours)

2. **Today: Implicit calculation**
   - Labour = `estimate_time` × number of staff child rows
   - No per-staff tracking

3. **After: Explicit tracking**
   - Labour = SUM(`task_assignments.planned_minutes`)
   - Per-staff planned AND actual time

4. **The formula stays the same**
   - `Planned Labour = Duration × Staff`
   - But now you can override per staff if needed

5. **Better reporting capabilities**
   - Planned vs actual comparison per staff
   - Variance analysis
   - Accurate overtime tracking

---

**Document End**
