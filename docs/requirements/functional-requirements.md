# Functional Requirements

## Overview

This document details the functional requirements of the system - what the system should do and the features it must provide to meet user needs.

---

## Table of Contents

- [Core Features](#core-features)
- [User Management](#user-management)
- [Business Logic](#business-logic)
- [Data Management](#data-management)
- [Integration Requirements](#integration-requirements)
- [Reporting and Analytics](#reporting-and-analytics)

---

## Core Features

### FR-001: Employee Authentication and Portal Integration
**Priority**: High  
**Status**: Required

The system shall integrate with the existing intranet portal's single-sign-on (SSO) mechanism using Central Authentication Service (CAS) for user authentication.

**Acceptance Criteria:**
- Employees authenticate through existing portal credentials
- No separate login required for VTS
- Session maintained across portal and VTS
- Automatic identification of employee role (Employee/Manager/Clerk/Admin)
- Support for re-authentication when session expires

**Related Use Cases:**  All use cases (precondition)

---

### FR-002: Vacation Time Request
**Priority**: High  
**Status**: Required

Employees shall be able to create, view, edit, withdraw, and cancel vacation time requests.

**Acceptance Criteria:**
- Employees can create new requests with date selection, hours, title, and description
- Employees can use visual calendar interface for date selection
- Employees can view requests from previous 6 months to 18 months in future
- Employees can edit pending requests (title, comments, dates)
- Employees can withdraw pending requests (before manager approval)
- Employees can cancel approved requests (future or recent past 5 business days)
- Email notification is sent automatically once the information is compiled and successfully validated
- System calculates total time spent automatically
- System displays current status of all requests

**Related Use Cases:** UC-01 (Manage Time) - Main Flow, Edit Pending Request, Withdraw Request, Cancel Approved Request

---

### FR-003: Manager Approval Workflow
**Priority**: High  
**Status**: Required

Managers shall be able to review and approve/deny subordinate vacation requests.

**Acceptance Criteria:**
- View list of pending approval requests from subordinates
- Access detailed information for each request
- Approve or deny requests with mandatory explanation for denials
- Email notification sent automatically upon approval/denial
- Managers can manage their own vacation time as employees
- Support for multiple managers per employee

**Related Use Cases:** UC-02 (Approve Request)

---

### FR-004: Comp Time Award Management
**Priority**: Medium
**Status**: Required

Managers shall be able to award personal leave time (comp time) to subordinates within system-defined limits.

**Acceptance Criteria:**
- Award comp time to direct subordinates
- System enforces maximum award limits
- Awards logged in activity log
- Employee balance updated immediately
- Notification sent to employee

**Related Use Cases:** UC-03 (Award Time)

---

## User Management

### FR-100: Role-Based Access Control
**Priority**: High
**Status**: Required

The system shall implement role-based access control for four distinct user roles.

**Acceptance Criteria:**
- Define roles: Employee, Manager, HR Clerk, System Administrator
- Managers inherit all Employee permissions
- HR Clerks have view access to employee personal data
- System Admins have technical system access only
- Role assignment managed through HR systems
- Audit logging for all privileged actions

**Related Use Cases:** All

---

### FR-101: Employee Profile and Leave Balance Display
**Priority**: High
**Status**: Required

The system shall display employee-specific information and vacation balances.

**Acceptance Criteria:**
- Display employee name and current date
- Show outstanding balance per leave category
- Display primary work location
- Show grant expiration dates
- Real-time balance calculations
- Access to previous 6 months and future 18 months of data

**Related Use Cases:** UC-01 (Manage Time)

---

### FR-102: HR Employee Record Management
**Priority**: High
**Status**: Required

HR Clerks shall manage employee vacation-related records.

**Acceptance Criteria:**
- View all employee vacation data
- Edit employee leave time allowances
- Set maximum comp time award limits per employee
- Assign primary work location
- Manage employee grant categories
- Interface with legacy HR systems for employee data synchronization

**Related Use Cases:** UC-04 (Edit Employee Record)

---

## Business Logic

### FR-200: Rules-Based Vacation Request Validation
**Priority**: High
**Status**: Required

The system shall validate all vacation requests against flexible, configurable business rules.

**Acceptance Criteria:**
- Validate against company-wide policies
- Validate against location-specific restrictions
- Validate against category-specific rules
- Support multiple restriction types:
    - Maximum consecutive days per leave type
    - Restrictions adjacent to holidays
    - Weekly/monthly hour limits
    - Minimum coworker coverage requirements
    - Date exclusions
    - Day-of-week restrictions
- Display all validation errors with clear explanations
- Prevent submission of invalid requests
- Real-time validation feedback

**Related Use Cases:** UC-01 (Manage Time) 

---

### FR-201: Location Management
**Priority**: High
**Status**: Required

HR Clerks shall manage work locations and their associated vacation rules.

**Acceptance Criteria:**
- Create, edit, and delete location records
- Assign location-specific restrictions
- Define location-specific holidays
- Associate employees with locations
- View all employees per location

**Related Use Cases:** UC-05 (Manage Locations)

---

### FR-202: Leave Category Management
**Priority**: High
**Status**: Required

HR Clerks shall manage leave categories and their rules.

**Acceptance Criteria:**
- Create, edit, and delete leave categories (e.g., vacation, sick, personal)
- Define category-specific restrictions
- Set default allowances per category
- Configure category expiration rules
- Associate grants with categories

**Related Use Cases:** UC-06 (Manage Leave Categories)

---

### FR-203: Grant Management
**Priority**: High
**Status**: Required

The system shall manage vacation time grants for employees.

**Acceptance Criteria:**
- Define hours available per calendar year per grant
- Set grant expiration dates
- Associate grants with categories
- Track grant usage
- Automatically deduct hours from appropriate grants
- Return hours when requests are canceled

**Related Use Cases:** UC-01 (Manage Time), UC-04 (Edit Employee Record)

---

### FR-204: Override Capability
**Priority**: Medium
**Status**: Required

HR Clerks shall be able to override rule-based rejections with full audit logging.

**Acceptance Criteria:**
- Override any system rejection
- Mandatory explanation for overrides
- Complete audit trail of override actions
- Override logged with clerk identity, timestamp, and reason
- Display override history

**Related Use Cases:** UC-07 (Override Leave Records)

---

### FR-205: Request State Management
**Priority**: High
**Status**: Required

The system shall manage vacation request lifecycle states.

**Acceptance Criteria:**
- Support states: Pending, Pending Approval, Approved, Denied, Withdrawn, Canceled
- Enforce state transition rules
- Track state change history
- Display current state to employees and managers
- Prevent invalid state transitions

**Related Use Cases:** UC-01 (Manage Time), UC-02 (Approve Request)

---

## Data Management

### FR-300: Activity Logging
**Priority**: High
**Status**: Required

The system shall maintain comprehensive activity logs for all transactions.

**Acceptance Criteria:**
- Log all vacation request creation, modification, approval, denial
- Log all HR override actions
- Log all grant modifications
- Include user identity, timestamp, action type, and affected records
- Logs stored securely and cannot be modified
- Retention period compliance

**Related Use Cases:** All

---

### FR-301: System Log Backup
**Priority**: High
**Status**: Required

System Administrators shall be able to back up and archive system logs.

**Acceptance Criteria:**
- Manual backup initiation
- Scheduled automatic backups
- Export logs in standard format
- Verify backup integrity
- Archive to secure storage
- Maintain backup retention schedule

**Related Use Cases:** UC-08 (Back Up System Logs)

---

### FR-302: Historical Data Access
**Priority**: High
**Status**: Required

The system shall provide access to historical vacation data.

**Acceptance Criteria:**
- Access previous 6 months of request data
- Access up to 18 months future requests
- View request history per employee
- Display state change audit trail
- Support historical reporting

**Related Use Cases:** UC-01 (Manage Time)

---

## Integration Requirements

### FR-400: Email Notification System
**Priority**: High
**Status**: Required

The system shall send automated email notifications for key events.

**Acceptance Criteria:**
- Notify managers when approval required
- Notify employees of approval/denial decisions
- Include clickable links to relevant system pages
- Notify managers of request withdrawals
- Notify managers of request cancellations
- Configurable email templates
- Support for email delivery failure handling

**Related Use Cases:** UC-01 (Manage Time), UC-02 (Approve Request)

---

### FR-401: HR Legacy System Integration
**Priority**: High
**Status**: Required

The system shall interface with HR department legacy systems.

**Acceptance Criteria:**
- Retrieve employee information from HR systems
- Sync employee data changes automatically
- Support batch updates
- Handle data transformation between systems
- Error handling for integration failures
- Maintain data consistency

**Related Use Cases:** UC-04 (Edit Employee Record)

---

### FR-402: Web Service Interface
**Priority**: Medium
**Status**: Required

The system shall provide a Web service interface for internal system queries.

**Acceptance Criteria:**
- RESTful API for vacation request summaries
- Query by employee ID
- Return current balances and request status
- Secure API authentication
- Rate limiting
- API documentation

**Related Use Cases:** External system integration

---

### FR-403: Portal Integration
**Priority**: High
**Status**: Required

The system shall integrate seamlessly with the existing intranet portal.

**Acceptance Criteria:**
- Accessible as portal module/extension
- Use portal navigation framework
- Consistent look and feel with portal
- Share authentication with portal
- Support portal's single-sign-on
- Breadcrumb navigation integration

**Related Use Cases:** All

---

## Reporting and Analytics

#### FR-500: Employee Dashboard

**Priority**: High  
**Status**: Required

Employees shall have a personalized home page displaying vacation information.

**Acceptance Criteria:**
- Current request summary (all states)
- Outstanding balances per category
- Pending approval indicators
- Recent activity
- Quick links to create new requests
- Manager section (for managers showing subordinate pending requests)
- Informational messages display
- Responsive design for various devices

**Related Use Cases:** UC-01 (Manage Time)

---

#### FR-501: Request Summary Reporting

**Priority**: Medium  
**Status**: Optional

The system should provide summary reports for managers and HR.

**Acceptance Criteria:**
- View requests by date range
- Filter by employee, location, category
- Export reports (CSV, PDF)
- Summary statistics
- Trend analysis

**Related Use Cases:** Manager and HR clerk workflows

---

# Requirements Traceability Matrix

| Requirement ID | Feature | Priority | Use Cases | Actor | Status |
|---------------|---------|----------|-----------|-------|---------|
| FR-001 | Portal SSO Authentication | High | All | All | Required |
| FR-002 | Vacation Request Management | High | UC-01 | Employee | Required |
| FR-003 | Manager Approval | High | UC-02 | Manager | Required |
| FR-004 | Comp Time Awards | Medium | UC-03 | Manager | Required |
| FR-100 | Role-Based Access | High | All | All | Required |
| FR-101 | Employee Profile Display | High | UC-01 | Employee | Required |
| FR-102 | HR Record Management | High | UC-04 | HR Clerk | Required |
| FR-200 | Rules-Based Validation | High | UC-01 | System | Required |
| FR-201 | Location Management | High | UC-05 | HR Clerk | Required |
| FR-202 | Category Management | High | UC-06 | HR Clerk | Required |
| FR-203 | Grant Management | High | UC-01, UC-04 | System | Required |
| FR-204 | HR Override | Medium | UC-07 | HR Clerk | Required |
| FR-205 | State Management | High | UC-01, UC-02 | System | Required |
| FR-300 | Activity Logging | High | All | System | Required |
| FR-301 | Log Backup | High | UC-08 | Sys Admin | Required |
| FR-302 | Historical Data | High | UC-01 | Employee | Required |
| FR-400 | Email Notifications | High | UC-01, UC-02 | System | Required |
| FR-401 | HR System Integration | High | UC-04 | System | Required |
| FR-402 | Web Service API | Medium | External | System | Required |
| FR-403 | Portal Integration | High | All | System | Required |
| FR-500 | Employee Dashboard | High | UC-01 | Employee | Required |
| FR-501 | Reporting | Medium | Various | Manager, HR | Optional |

---

# Requirements Prioritization

## Must Have (Critical - Release 1.0)

- Portal SSO Authentication (FR-001)
- Vacation Request Management (FR-002)
- Manager Approval Workflow (FR-003)
- Role-Based Access Control (FR-100)
- Employee Profile Display (FR-101)
- HR Record Management (FR-102)
- Rules-Based Validation (FR-200)
- Location Management (FR-201)
- Category Management (FR-202)
- Grant Management (FR-203)
- State Management (FR-205)
- Activity Logging (FR-300)
- Log Backup (FR-301)
- Historical Data Access (FR-302)
- Email Notifications (FR-400)
- HR System Integration (FR-401)
- Portal Integration (FR-403)
- Employee Dashboard (FR-500)

## Should Have (High Priority - Release 1.1)

- Comp Time Awards (FR-004)
- HR Override Capability (FR-204)
- Web Service API (FR-402)

## Could Have (Medium Priority - Release 2.0)

- Advanced Reporting (FR-501)
- Mobile-specific optimizations
- Bulk request operations
- Calendar integration (Google Calendar, Outlook)

## Won't Have (Low Priority / Future Releases)

- Multi-language support
- Advanced analytics dashboard
- Machine learning for request predictions
- Integration with external HR systems (non-legacy)
- Mobile native applications
- Vacation time trading between employees
- Team vacation calendar view
- Automated vacation suggestions based on patterns
---
