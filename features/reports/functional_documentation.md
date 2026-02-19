# Reports & Incident Management
## Functional Requirements Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-FRS-REPT-001 |
| **Version** | 1.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Product Team |
| **Classification** | Internal |

---

## Executive Summary

The **Reports & Incident Management** module enables field staff and managers to document, track, and resolve site incidents including disease outbreaks, infrastructure damage, pest infestations, and environmental events. Reports are geo-tagged on the property map and feed into Maya's AI disease prediction models.

This document defines the functional requirements for incident reporting, follow-up tracking, status workflows, and integration with mapping and AI systems.

**Key Business Outcomes:**
- **Rapid Response**: Immediate visibility of issues enables faster intervention.
- **Pattern Recognition**: Historical incident data reveals recurring problem areas.
- **AI Improvement**: Disease outbreak reports train Maya's prediction models for site-specific accuracy.
- **Audit Compliance**: Complete incident records support insurance and regulatory requirements.

---

## 1. Scope & Objectives

### 1.1 In Scope
- Incident report creation with geo-location
- Photo attachment and documentation
- Severity classification and status tracking
- Follow-up action logging
- Map marker visualization
- Excel/CSV export
- WhatsApp integration for field reporting

### 1.2 Out of Scope
- Automated incident detection from sensors
- Work order generation (handled by Task module)
- Root cause analysis (future enhancement)

### 1.3 Target Users
| Role | Primary Use Case |
|:---|:---|
| **Grounds Staff** | Initial incident reporting from field |
| **Superintendent** | Review, prioritize, assign follow-up |
| **Agronomist** | Disease pattern analysis |
| **Operations Manager** | Incident metrics and reporting |

---

## 2. Report Creation

### 2.1 Report Form Fields

| Field | Type | Required | Description |
|:---|:---|:---|:---|
| **Date** | Date picker | Yes | Date incident observed |
| **Type** | Dropdown | Yes | Incident category (configurable) |
| **Course/Site** | Dropdown | Yes | Parent location |
| **Hole/Pitch** | Dropdown | No | Specific sub-location |
| **Area/Zone** | Multi-select | Yes | Affected zones |
| **Severity** | Radio (Low/Med/High) | No | Impact level |
| **Status** | Dropdown | Yes | Current state |
| **Comments** | Text area | No | Free-text description |
| **Image** | File upload | No | Photo documentation |
| **Map Marker** | Map click | No | GPS coordinates |

### 2.2 Default Incident Types

| Category | Types |
|:---|:---|
| **Disease** | Anthracnose, Dollar Spot, Microdochium Patch, Fairy Ring |
| **Pest Damage** | Leatherjacket, Bird Damage, Boar Damage, Mole Damage |
| **Infrastructure** | Drainage, Irrigation, Bunker Washout, Fallen Tree |
| **Environmental** | Dry Spot, Local Flood |

### 2.3 Custom Incident Types

Site administrators can create custom incident types via the "List of Incidents" management panel. Custom types:
- Automatically appear in the Type dropdown
- Can be activated/deactivated
- Persist across all users in the tenant

---

## 3. Severity Classification

| Level | Icon | Criteria |
|:---|:---|:---|
| **Low** | 🟡 Yellow | Cosmetic impact, no immediate action needed |
| **Medium** | 🟠 Orange | Notable damage, action within 1 week |
| **High** | 🔴 Red | Critical issue, immediate intervention required |

---

## 4. Status Workflow

### 4.1 Status States

| Status | Description |
|:---|:---|
| **Reported** | Initial submission, awaiting review |
| **In Progress** | Action being taken |
| **Resolved** | Issue addressed successfully |
| **Closed** | Complete and archived |

### 4.2 Status Transitions

```
┌──────────┐    ┌────────────┐    ┌──────────┐    ┌────────┐
│ Reported │───▶│ In Progress│───▶│ Resolved │───▶│ Closed │
└──────────┘    └────────────┘    └──────────┘    └────────┘
      │                                │
      └────────────────────────────────┘
                (Direct close if minor)
```

---

## 5. Follow-Up Tracking

### 5.1 Follow-Up Actions

Each report can have multiple follow-up entries:

| Field | Description |
|:---|:---|
| Action Date | When follow-up occurred |
| Description | What was done |
| Status Update | New status (if changed) |
| User | Who performed action |

### 5.2 Schedule Integration

Follow-up actions appear automatically in the Schedule view when a staff member is assigned. This provides a unified view of incident response alongside regular operations.

**How it works:**
1. When a follow-up is created with an assigned staff member, it appears in their Schedule for the specified date
2. Staff can update follow-up status directly from the Schedule page
3. Status changes sync back to the Reports module

**Display in Schedule:**
- Follow-ups are shown with the incident type as the task title
- They are visually distinguished from regular schedule tasks
- The assigned greenkeeper sees them alongside their daily tasks

> [!NOTE]
> Follow-ups use a separate notification system (`task_push_notification` table) rather than creating entries in the main `task` table. This allows for independent status tracking while maintaining Schedule visibility.

---

## 6. Map Integration

### 6.1 Marker Placement

When creating a report, users can:
1. Click "Set Marker" button
2. Click on the satellite map to place pin
3. Coordinates are automatically captured

### 6.2 Map Visualization

All reports appear as markers on the Maps module:
- Marker color indicates severity
- Clicking marker shows incident details
- Markers can be filtered by type, date, status

---

## 7. Reporting & Export

### 7.1 Table Columns

| Column | Description |
|:---|:---|
| Photo | Thumbnail preview |
| Date | Report date |
| Type | Incident category |
| Course | Location |
| Area | Zones affected |
| Hole | Specific location |
| Severity | Impact level |
| Comments | Description |
| Status | Current state |
| Actions | Edit / Delete |

### 7.2 Filtering

| Filter | Options |
|:---|:---|
| Search | Free-text search |
| Type | Dropdown of incident types |
| Course | Location filter |
| Hole | Specific location |

### 7.3 Column Configuration

Users can show/hide columns via the "Column Settings" menu.

### 7.4 Export Options

| Format | Contents |
|:---|:---|
| **Excel** | All visible columns, all records |
| **Export Data** | Full dataset for external analysis |

---

## 8. WhatsApp Integration

### 8.1 Maya Bot Reporting

Field staff can report incidents via WhatsApp:
1. Send photo to Maya bot
2. Bot prompts for type, location, severity
3. Report is created automatically
4. Appears in web interface with WhatsApp source tag

This enables rapid field reporting without needing web/app access.

---

## 9. AI Integration

### 9.1 Disease Prediction Improvement

Reported disease outbreaks contribute to Maya's AI models:
- Location data trains site-specific patterns
- Severity helps calibrate risk thresholds
- Historical data improves forecast accuracy

> [!TIP]
> Regularly reporting disease outbreaks significantly improves Maya's prediction accuracy for your specific site conditions.

---

## 10. Acceptance Criteria

1. Reports can be created with all required fields and saved successfully.
2. Photo uploads attach correctly and display in report list.
3. Map markers appear at correct GPS coordinates.
4. Status changes update immediately in report list.
5. Custom incident types appear in dropdown after creation.
6. Excel export includes all visible columns and filtered data.
7. WhatsApp-submitted reports appear in web interface.

---

**Document End**
