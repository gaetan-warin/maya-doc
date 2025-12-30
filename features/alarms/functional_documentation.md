# Alarms & Notifications
## Functional Requirements Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-FRS-ALRM-001 |
| **Version** | 1.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Product Team |
| **Classification** | Internal |

---

## Executive Summary

The **Alarms & Notifications** module enables proactive alerting based on weather metrics, agronomic conditions, and incident reports. Users configure threshold-based or event-based alarms that trigger notifications via Email and/or WhatsApp, ensuring timely response to critical site conditions.

This document defines the functional requirements for alarm configuration, notification delivery, and alarm history tracking. It integrates with Weather, Agronomy, and Reports modules to provide comprehensive site monitoring.

**Key Business Outcomes:**
- **Proactive Response**: Immediate alerts enable rapid intervention before conditions worsen.
- **Reduced Risk**: Threshold monitoring prevents disease outbreaks and equipment failures.
- **Operational Efficiency**: Automated notifications reduce manual monitoring burden.
- **Flexibility**: Multi-channel delivery ensures alerts reach staff wherever they are.

---

## 1. Scope & Objectives

### 1.1 In Scope
- Metric-based alarm configuration (weather, soil, disease risk)
- Incident-based alarm configuration (triggered by report submission)
- Notification via Email and WhatsApp
- Alarm history logging
- Spraying notification reminders

### 1.2 Out of Scope
- SMS notifications (future enhancement)
- Push notifications to mobile app
- Escalation workflows (if unacknowledged)
- Integration with external alerting systems (PagerDuty, etc.)

### 1.3 Target Users
| Role | Primary Use Case |
|:---|:---|
| **Superintendent** | Configure alarms for site conditions |
| **Agronomist** | Set disease risk thresholds |
| **Operations Manager** | Monitor alarm history |
| **Field Staff** | Receive and act on notifications |

---

## 2. Alarm Types

### 2.1 Metric Alarms

Metric alarms trigger when measured or forecasted values cross configured thresholds.

| Category | Available Metrics |
|:---|:---|
| **Disease Risk** | Anthracnose Risk, Dollar Spot Risk, Microdochium Risk, Pythium Risk |
| **Disease Forecast** | Anthracnose Risk (Forecast), Dollar Spot Risk (Forecast), etc. |
| **Weather** | Daily Rainfall, Humidity, Air Temperature, Wind Speed |
| **Soil** | Soil Moisture, Soil Temperature, Soil Oxygen |
| **Agronomy** | ET (Evapotranspiration), Leaf Wetness, DLI (Daily Light Integral) |

> [!TIP]
> "Risk" alarms use current sensor data, while "Risk (Forecast)" alarms use predicted values for proactive planning.

### 2.2 Incident Alarms

Incident alarms trigger when specific incident types are reported through the Reports module.

| Incident Category | Examples |
|:---|:---|
| **Disease** | Anthracnose, Dollar Spot, Fairy Ring, Microdochium Patch |
| **Pest Damage** | Bird Damage, Boar Damage, Leatherjacket, Mole Damage |
| **Infrastructure** | Drainage, Irrigation, Bunker Washout, Fallen Tree |
| **Environmental** | Dry Spot, Local Flood, Red Thread, Weeds Patch |

---

## 3. Alarm Configuration

### 3.1 Metric Alarm Form

| Field | Type | Description |
|:---|:---|:---|
| **Rule Type** | Toggle | Metric / Incident |
| **Action Name** | Dropdown | Select metric to monitor |
| **Type (Condition)** | Dropdown | `< is less than`, `= is equal to`, `> is greater than` |
| **Value** | Number | Threshold value |

### 3.2 Incident Alarm Form

| Field | Type | Description |
|:---|:---|:---|
| **Rule Type** | Toggle | Metric / Incident |
| **Incident Name** | Dropdown | Select incident type |
| **Rule** | Fixed | "When reported" |

### 3.3 Alarm Actions

| Action | Description |
|:---|:---|
| **Create** | Add new alarm via [+] button |
| **Edit** | Modify existing alarm |
| **Delete** | Remove alarm |
| **Toggle Status** | Enable/Disable without deleting |

---

## 4. Notification Channels

### 4.1 Email Notifications

- Sent to the email address associated with the user's login
- Includes alarm name, triggered value, timestamp
- Configured in user account settings

### 4.2 WhatsApp Notifications

- Requires phone number with country code in account settings
- Delivered via Maya WhatsApp bot
- Includes alarm details with quick action links

### 4.3 Notification Settings

Notification preferences are managed globally in **Settings > My Account**:

| Setting | Description |
|:---|:---|
| Email Address | Verified email for notifications |
| Phone Number | WhatsApp-enabled number with country code |
| Notification Preferences | Toggle Email/WhatsApp per user |

---

## 5. Alarms Page Layout

### 5.1 Active Alarms Section

| Column | Description |
|:---|:---|
| Name | Metric or Incident name |
| Type | Condition (e.g., "> is greater than") |
| Value | Threshold value |
| Action | Edit / Delete buttons |

### 5.2 Alarms History Section

| Column | Description |
|:---|:---|
| Datetime | When alarm triggered |
| Alarm Type | Metric or Incident name |
| Device Name | Sensor/source that triggered |
| Notification Method | Email / WhatsApp |
| Value | Actual value at trigger time |

### 5.3 Spraying Notifications Section

A separate component for spraying reminders:
- Configure reminders before scheduled spraying tasks
- Notification timing (hours before)
- Linked to spraying calendar

---

## 6. Alarm Workflow

### 6.1 Metric Alarm Trigger Flow

```
┌──────────────┐    ┌─────────────┐    ┌────────────┐    ┌────────────┐
│ Sensor Data  │───▶│ Threshold   │───▶│ Generate   │───▶│ Notify     │
│ Received     │    │ Check       │    │ Alarm      │    │ User       │
└──────────────┘    └─────────────┘    └────────────┘    └────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │ Log to      │
                    │ History     │
                    └─────────────┘
```

### 6.2 Incident Alarm Trigger Flow

```
┌──────────────┐    ┌─────────────┐    ┌────────────┐    ┌────────────┐
│ Report       │───▶│ Check       │───▶│ Generate   │───▶│ Notify     │
│ Submitted    │    │ Alarm Config│    │ Alarm      │    │ User       │
└──────────────┘    └─────────────┘    └────────────┘    └────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │ Log to      │
                    │ History     │
                    └─────────────┘
```

---

## 7. Integration Points

| System | Integration Type | Data Flow |
|:---|:---|:---|
| **Weather Module** | Inbound | Metric values for threshold checks |
| **Agronomy Module** | Inbound | Disease risk calculations |
| **Reports Module** | Inbound | Incident submissions |
| **Spraying Calendar** | Reference | Task timing for reminders |
| **User Settings** | Reference | Notification preferences |

---

## 8. Acceptance Criteria

1. Metric alarms can be created with action name, condition, and threshold value.
2. Incident alarms can be created with incident type selection.
3. Alarms trigger when threshold/condition is met.
4. Notifications are delivered via configured channels (Email/WhatsApp).
5. All triggered alarms are logged in Alarms History with timestamp.
6. Alarms can be edited and deleted from the active list.
7. Spraying notifications remind users based on configured timing.

---

**Document End**
