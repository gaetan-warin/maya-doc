# Schedule API Documentation

**Core 2.0 Dev URL:** `https://api.mayaglobal-dev.io`

---

## Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v2/createSchedule` | Create Schedule |
| GET | `/api/v2/getScheduleIntialData` | Get Initial Data for Page Load |
| GET | `{{baseUrl}}/{{apiVersion}}/schedules` | Tasks and Sprayings Data for Calendar |
| DELETE | `/schedule/{mergeKey}` | Delete Schedule (by merge_key) |
| DELETE | `/schedule/task/{taskId}` | Delete Single Task Schedule |
| PUT | `/api/v2/update-schedule` | Update task based on merge_key |

---

## Merge Key Generating Process

1. Concatenate site/hole IDs + modificationDate + actionId + currentTime
2. Determine TaskType:
   - `"site-based"` → if holes are selected
   - `"tenant-based"` → if only sites are selected

---

(Archived from Notion on 2025-12-22)
