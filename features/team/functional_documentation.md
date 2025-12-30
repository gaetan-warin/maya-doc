# Team Management
## Functional Requirements Specification

---

### Document Control
| Attribute | Value |
|:---|:---|
| **Document ID** | MAYA-FRS-TEAM-001 |
| **Version** | 1.0 |
| **Last Updated** | December 30, 2025 |
| **Status** | Approved |
| **Owner** | Product Team |
| **Classification** | Internal |

---

## Executive Summary

The **Team Management** module provides comprehensive human resource management capabilities including staff availability tracking, leave management, user account administration, and role-based permission configuration. It encompasses two integrated areas: **My Team** (operational staff management) and **User Management** (administrative account control).

This document defines the functional requirements for leave requests, working hour tracking, team configuration, user creation, and permission role management. It integrates with Schedule and Reports modules for unified workforce planning.

**Key Business Outcomes:**
- **Workforce Visibility**: Real-time view of staff availability and leave status.
- **Access Control**: Granular permissions ensure data security.
- **Operational Efficiency**: TOIL and overtime tracking streamlines payroll.
- **Compliance**: Leave history and approval workflow for audit trails.

---

## 1. Scope & Objectives

### 1.1 In Scope
- Staff leave request and approval workflow
- TOIL (Time Off In Lieu) and overtime tracking
- Team (group) configuration with site assignment
- User account creation and management
- Role-based permission configuration
- Mobile access management

### 1.2 Out of Scope
- Payroll integration (future enhancement)
- External HR system synchronization
- Biometric attendance tracking

### 1.3 Target Users
| Role | Primary Use Case |
|:---|:---|
| **HR Manager** | Leave approval, user management |
| **Team Manager** | Staff availability oversight |
| **Administrator** | Permission configuration |
| **Staff Member** | Leave requests, availability view |

---

## 2. My Team (Staff Management)

### 2.1 Overview

My Team provides operational staff management with focus on availability, leave, and working hours.

### 2.2 Leave Request Management

#### 2.2.1 Request Tabs

| Tab | Description |
|:---|:---|
| **Time Off Request** | Pending requests awaiting action |
| **Upcoming TOIL/Overtime** | Scheduled compensatory time |
| **Upcoming Time Off** | Approved future leave |
| **History** | Past leave records |
| **Archived** | Closed/rejected requests |

#### 2.2.2 Request Table Columns

| Column | Description |
|:---|:---|
| Staff Member | Employee name |
| Number of Days | Leave duration |
| From | Start date |
| To | End date |
| Reason | Leave type |
| Comment | Additional notes |
| Action | Approve/Reject buttons |

#### 2.2.3 Leave Types

| Type | Color Code | Description |
|:---|:---|:---|
| Annual Leave | Blue | Planned vacation |
| Sick Leave | Yellow | Illness absence |
| Absent | Orange | Unauthorized absence |
| Present | Green | Normal working day |
| Public Holiday | Purple | Bank holiday |

### 2.3 Staff Availability Overview

#### 2.3.1 View Modes

| Mode | Description |
|:---|:---|
| **Overview** | Calendar-based status view |
| **Hours** | Working hours breakdown |

#### 2.3.2 Staff Table Columns

| Column | Description |
|:---|:---|
| Staff Name | Employee name |
| Overtime Hours | Accumulated overtime |
| TOIL | Time off in lieu balance |
| Balance | Net availability |

### 2.4 Visual Calendar

Color-coded monthly view showing:
- Daily status for each staff member
- Navigation between months
- Click to view individual staff profile

### 2.5 Staff Profile Actions

| Action | Description |
|:---|:---|
| View Details | Open staff profile card |
| Edit Availability | Update weekly schedule |
| Add Leave | Create leave request |
| View History | Past leave and TOIL records |

---

## 3. Team Configuration

### 3.1 Overview

Teams (Groups) organize staff members and define default working parameters.

### 3.2 Team Table Columns

| Column | Description |
|:---|:---|
| ID | Team identifier |
| Avatar | Team icon |
| Name | Team name |
| Default Days Off | Annual leave allowance |
| Working Hours | Hours per day |
| Site | Associated location(s) |
| Settings | Edit/Delete actions |

### 3.3 Team Form Fields

| Field | Type | Description |
|:---|:---|:---|
| Team Name | Text | Team identifier |
| Sites | Checkbox | Associated locations |
| Days Off per Year | Number | Default annual leave |
| Work Hours per Day | Number | Standard working hours |

---

## 4. User Management

### 4.1 Overview

User Management handles account creation, role assignment, and access configuration.

### 4.2 User Table Columns

| Column | Description |
|:---|:---|
| ID | User identifier |
| Avatar | Profile picture |
| First Name | Given name |
| Last Name | Family name |
| Role | Admin / Viewer |
| Permission Role | Custom permission set |
| Is Staff | Boolean (for scheduling) |
| Mobile Access | App access status |
| Team Name | Assigned team |
| Settings | Edit/Delete actions |

### 4.3 User Form Fields

| Field | Type | Required | Description |
|:---|:---|:---|:---|
| First Name | Text | Yes | Given name |
| Last Name | Text | Yes | Family name |
| Email | Email | Yes | Login email |
| Login Method | Dropdown | Yes | Authentication type |
| Primary Role | Dropdown | Yes | Admin/Viewer |
| Permission Role | Dropdown | No | Custom permissions |
| Is Staff | Toggle | No | Appears in scheduling |
| Team | Dropdown | No | Team assignment |
| Team Manager | Toggle | No | Manager privileges |
| Mobile Access | Toggle | No | App access (plan-dependent) |

### 4.4 Password Management

| Action | Description |
|:---|:---|
| Set Password | Initial password creation |
| Change Password | Password update |
| Reset Password | Admin-initiated reset |

---

## 5. Permission Roles

### 5.1 Overview

Custom permission roles define granular access levels across all system modules.

### 5.2 Permission Categories

| Category | Permissions |
|:---|:---|
| **Inventory** | Purchase requests, stock, invoices, deliveries, tax rates |
| **Fleet** | Machinery hours, maintenance, fuel, precheck lists |
| **Nutritional Inputs** | Create, edit, delete |
| **Water** | Create, edit, delete |
| **Reports** | View, export |
| **Agronomy** | Disease models, recommendations |
| **Planner** | Fertilisation, budget, routines |
| **Schedule** | Task management, assignments |

### 5.3 Permission Levels

| Level | Description |
|:---|:---|
| **None** | No access |
| **View** | Read-only access |
| **Create** | Can create new entries |
| **Edit** | Can modify existing entries |
| **Delete** | Can remove entries |
| **Full** | All permissions |

### 5.4 Role Form

| Field | Description |
|:---|:---|
| Role Name | Custom role name |
| Permission Grid | Module-by-action checkboxes |

---

## 6. Mobile Access

### 6.1 Overview

Mobile access is controlled per-user and subject to subscription limits.

### 6.2 Access Control

| Feature | Description |
|:---|:---|
| Mobile Access Toggle | Enable/disable per user |
| Access Count | Current usage vs. plan limit |
| Plan Upgrade | Link to subscription settings |

---

## 7. Integration Points

| System | Integration Type | Data Flow |
|:---|:---|:---|
| **Schedule Module** | Consumer | Staff availability |
| **Reports Module** | Consumer | Labor hours |
| **Authentication** | Provider | Login credentials |
| **Subscription** | Reference | Mobile access limits |

---

## 8. Acceptance Criteria

1. Leave requests can be created, approved, and rejected.
2. Staff availability calendar displays correct status colors.
3. Teams can be created with site assignment.
4. Users can be created with role and team assignment.
5. Custom permission roles can be created with granular settings.
6. Mobile access respects subscription limits.
7. TOIL and overtime tracking maintains accurate balances.

---

**Document End**
