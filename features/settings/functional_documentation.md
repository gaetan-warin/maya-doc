# Settings Module
## Functional Requirements Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-FRS-SETT-001 |
| **Version** | 1.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Product Team |
| **Classification** | Internal |

---

## Executive Summary

The **Settings Module** provides centralized configuration management for all system aspects including user profiles, site setup, device management, module-specific preferences, and integrations. It encompasses 15 distinct configuration sections covering personal settings, operational parameters, hardware configuration, and third-party integrations.

This document defines the functional requirements for system configuration across all operational domains. It integrates with all other modules to provide unified preference management.

**Key Business Outcomes:**
- **Centralized Control**: Single location for all system configuration.
- **Personalization**: User-specific dashboard and notification preferences.
- **Operational Efficiency**: Preconfigured defaults reduce repetitive setup.
- **Integration Ready**: API token management for external systems.

---

## 1. Scope & Objectives

### 1.1 In Scope
- User profile and account management
- Site and zone configuration
- Device and sensor management
- Module-specific settings (Agronomy, Schedule, Fleet, etc.)
- Notification preferences
- API token management
- Measurement unit preferences

### 1.2 Out of Scope
- Subscription/billing management
- System administration (database, server)
- White-label customization

### 1.3 Target Users
| Role | Primary Use Case |
|:---|:---|
| **All Users** | Personal preferences |
| **Administrator** | System configuration |
| **Superintendent** | Site and schedule settings |
| **IT Manager** | API integrations |

---

## 2. My Account

### 2.1 Overview

Personal profile management and notification preferences.

### 2.2 Profile Fields

| Field | Type | Description |
|:---|:---|:---|
| First Name | Text | User's given name |
| Last Name | Text | User's family name |
| Email | Email | Login and notification email |
| Profile Photo | Image | Avatar upload |

### 2.3 WhatsApp Notifications

| Field | Description |
|:---|:---|
| Country Code | Dropdown selector |
| Phone Number | WhatsApp-enabled number |

### 2.4 Work Statistics

| Metric | Description |
|:---|:---|
| Total Days Worked | Year-to-date count |
| Total Days Off | Year-to-date leave taken |

---

## 3. My Business

### 3.1 Overview

Display of business identity and type.

### 3.2 Display Fields

| Field | Description |
|:---|:---|
| Course/Club Name | Business name (e.g., "Maya Demo") |
| Business Type | Golf, Football, Landscaping, etc. |

---

## 4. Change Password

### 4.1 Overview

Secure password update functionality.

### 4.2 Form Fields

| Field | Description |
|:---|:---|
| Old Password | Current password verification |
| New Password | New password input |
| Confirm Password | Password confirmation |

### 4.3 Password Requirements

- Minimum 10 characters
- At least one digit
- At least one lowercase letter
- At least one uppercase letter
- At least one special character

---

## 5. Setup My Site

### 5.1 Overview

Site location and zone configuration with three sub-tabs.

### 5.2 Tab: Locate Your Club

| Field | Description |
|:---|:---|
| Business Name | Site name |
| Country | Country selection |
| Latitude | GPS latitude |
| Longitude | GPS longitude |
| Map Interface | Interactive Google Maps |

### 5.3 Tab: Set Up Your Site

| Column | Description |
|:---|:---|
| Name | Site/course name |
| Total Surface | Area in m² |
| Actions | Edit, Delete |

### 5.4 Tab: Preferred Work Zones

Toggle list for zone types:
- Fairways
- Greens
- Rough
- Tee
- Bunkers
- Infrastructure
- Other zones

---

## 6. My Connected Devices

### 6.1 Overview

Hardware device management and status monitoring.

### 6.2 Status Legend

| Color | Status |
|:---|:---|
| Green | Fully Functional |
| Yellow | Partial Metric Delay |
| Red | No Metric Received |
| Gray | Voltage Failure |

### 6.3 Device Table

| Column | Description |
|:---|:---|
| Device Name | Hardware identifier |
| Display Name | User-friendly label |
| Assigned Zone | Mapped location |
| Actions | Edit |

### 6.4 Device Types

- Soil Sensors
- Weather Stations
- ET Data devices

---

## 7. Agronomy Settings

### 7.1 Insight Settings

Configure 6 dashboard insight cards:
- Dollar Spot risk
- GDD (Growing Degree Days)
- Water Balance
- Microdochium risk
- And more...

### 7.2 Nutritional Input Settings

| Setting | Options |
|:---|:---|
| Task Status Default | Completed / To-Do |
| Mode | Standard / Advanced |
| Default Dissolution | L per hectare |
| Tank Volume | Liters |
| Precision Fertilisation | Standard / Advanced |

---

## 8. Dashboard Settings

### 8.1 Link Devices

Map metrics to hardware:
- ET sensor
- Rainfall gauge
- Wind speed monitor
- Soil moisture sensors

### 8.2 Disease Settings

Select up to 4 disease models to monitor:
- Fusarium
- Anthracnose
- Dollar Spot
- Microdochium

### 8.3 Metric Settings

Configure up to 4 performance metrics for dashboard display.

---

## 9. Portable Soil Sensor Settings

### 9.1 VWC Thresholds

Configure Volumetric Water Content ranges:

| Range | Description |
|:---|:---|
| Very Low | Threshold value |
| Low | Threshold value |
| Optimal | Threshold value |
| High | Threshold value |

---

## 10. Schedule Settings

### 10.1 Mowing Parameters

| Feature | Description |
|:---|:---|
| Mowing Parameters | Toggle on/off |
| Rotation Tracker | Toggle on/off |

### 10.2 Mowing Directions

| Column | Description |
|:---|:---|
| Type | Direction type |
| Color | Visual indicator |

### 10.3 Mowing Patterns

Pattern options:
- Parallel
- Diamond
- Stripes
- Custom

### 10.4 Leave Types

Manage staff leave labels:
- Absent
- Annual Leave
- Sick Leave
- Custom types

### 10.5 Public Holidays

| Column | Description |
|:---|:---|
| Date | Holiday date |
| Name | Holiday name |
| Actions | Edit, Delete |

---

## 11. Water Settings

### 11.1 Notifications

| Setting | Description |
|:---|:---|
| Reminder Time | Preferred notification time |

### 11.2 Water Inflow/Outflow

| Column | Description |
|:---|:---|
| Name | Source name |
| Source Type | Inflow / Outflow |
| Measurement Type | Meter Reading / Daily Consumption |
| Serial Number | Hardware ID |
| Status | Active/Inactive |

---

## 12. Inventory Settings

### 12.1 VAT & Tax

| Column | Description |
|:---|:---|
| Name | Tax name |
| Rate | Percentage |
| Actions | Edit, Delete |

### 12.2 Stock Notifications

| Setting | Options |
|:---|:---|
| Low Stock Alerts | Enable/Disable |
| Reminder Frequency | Every 7 days, Every 30 days, Initial only |

---

## 13. Fleet Settings

### 13.1 Notifications

| Setting | Description |
|:---|:---|
| Machine Failure Alerts | Enable/Disable |
| Fuel Usage Tracking | Enable/Disable |

### 13.2 Tags Management

Manage machinery tags for categorization.

---

## 14. Report Settings

### 14.1 Options

| Setting | Description |
|:---|:---|
| Display Follow-up Form | Toggle |
| Display on Digital Whiteboard | Toggle |

---

## 15. API Token Settings

### 15.1 Overview

External integration management through API tokens.

### 15.2 Token Table

| Column | Description |
|:---|:---|
| Name | Token name |
| Token | API key (masked) |
| Scope | Access level |
| Created At | Creation timestamp |
| Expiration Date | Validity period |
| Status | Active/Revoked |

### 15.3 Token Actions

| Action | Description |
|:---|:---|
| Create | Generate new token |
| Copy | Copy token to clipboard |
| Revoke | Disable token |
| Delete | Remove token |

---

## 16. General Settings

### 16.1 Site & Station

| Setting | Description |
|:---|:---|
| Weather Station Height | Meters |
| Landscape Type | Terrain classification |

### 16.2 Alarm Notification Channels

| Channel | Description |
|:---|:---|
| Email | Toggle |
| WhatsApp | Toggle |
| Push Notifications | Toggle |

### 16.3 Measurement Units

| System | Units |
|:---|:---|
| Metric | km, m, cm, kg, ha |
| Imperial | mi, yd, ft, ga, a |

---

## 17. Integration Points

| System | Integration Type | Data Flow |
|:---|:---|:---|
| **All Modules** | Consumer | Configuration preferences |
| **External APIs** | Provider | Token-based access |
| **Mobile App** | Consumer | Notification preferences |
| **Hardware** | Reference | Device configuration |

---

## 18. Acceptance Criteria

1. Profile updates persist and reflect across sessions.
2. Password change enforces security requirements.
3. Site location saves with correct GPS coordinates.
4. Device assignments map to correct zones.
5. Dashboard settings customize widget display.
6. Schedule settings affect task calendar behavior.
7. API tokens can be created with scoped access.
8. Measurement unit toggle affects all numeric displays.

---

**Document End**
