# Fleet Management

## Fleet Management 1.1
**Machine Add Process**
Initial load calls `getMachineData` to populate brands, models, serials, and fuel types. Based on model selection, the category and fuel type are determined. Saving is a two-step process: save basic machine data via 'Add machine' API, then add machine time via 'Add machine time' API using the returned machine ID. Machine names follow the format "[Model Name]_[Machine Number]". Brand and Model addition pop-ups are provided if they aren't in the list.

**Maintenance Add Process**
Triggered by the 'Add New Maintenance' button. A pop-up captures Maintenance name, Machine (from `getMachineData`), Status (urgent/minor/done/upcoming), Date, Assigned User, Created User, Cost, and Hours in Use. Data is saved to `machine_maintenance` and `machine_time` tables.

## Fleet Management 1.3 - Inspection section
**Machine Incident (POST | PATCH)** API captures incident details like severity, status, and comments, linking them to machines, tasks, or spraying sessions.

## Fleet Management 1.4 & 2.1
Machine list at bottom. APIs for getting all maintenance by machine (odata filter), getting machine data by ID, and patching machine parts.

## Fleet Management 2.2 - Recurring Maintenance
To create a rule, the recurrence API is called first, followed by the maintenance API. Includes APIs to Add Recurrence, Toggle Recurrence (enable/disable), Deactivate Maintenance, and retrieve all active maintenance for a tenant.

## Fleet Management 4 & 5 - Display and Alarms
`getMaintenanceDisplay` retrieves maintenance records based on status filters (urgent, minor, upcoming, due). `notifyIncident` triggers notifications (Email/Whatsapp) upon incident report. Settings page allows activating these notifications via User PATCH API.

(Archived from Notion on 2025-12-22)
