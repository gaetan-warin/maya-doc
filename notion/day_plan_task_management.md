# Day Plan Task Management

## Day Plan Task Creation

The system supports comprehensive task creation with the following features:

### 1. Multi-Site Selection
- Parent Sites
- Child Sites
- Multiple Selection

### 2. Staff and Machinery Assignment
- Assign one or more staff members
- Assign machinery/equipment

### 3. Time Management
- **AM** - Morning slot
- **PM** - Afternoon slot
- **Overtime** - Extra hours beyond schedule
- Natural language date parsing

### 4. API Integration
**Endpoint:** `POST /api/v2/createSchedule`

---

## Task Status Management

| Status | `running` | `task_start` | `task_end` | Description |
|--------|-----------|--------------|------------|-------------|
| `todo` | 0 | null | null | Task not started |
| `in_progress` | 0 | value | null | Task started but not completed |
| `completed` | 1 | value | value | Task finished |
| `not_completed` | 0 | value | value | Task stopped without completion |

---

(Archived from Notion on 2025-12-22)
