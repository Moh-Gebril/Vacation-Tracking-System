# System Constraints

## Overview

This document details the constraints and limitations that apply to the Vacation Tracking System (VTS) - the boundaries within which the system must be designed, developed, and operated.

---

## Table of Contents

- [Technical Constraints](#technical-constraints)
- [Business Constraints](#business-constraints)
- [Organizational Constraints](#organizational-constraints)
- [Environmental Constraints](#environmental-constraints)
- [Resource Constraints](#resource-constraints)
- [Legal and Regulatory Constraints](#legal-and-regulatory-constraints)
- [Integration Constraints](#integration-constraints)
- [User Interface Constraints](#user-interface-constraints)

---

## Technical Constraints

### CON-001: Technology Stack Mandate

**Type**: Technical/Organizational  
**Impact**: High

The system must be built using the organization's approved technology stack.

**Constraints:**
- Must use Node.js LTS version (currently 18.x or higher)
- Must use NestJS framework for backend development
- Must use TypeScript (no plain JavaScript)
- Must use PostgreSQL or MySQL for relational data (organization's licensed databases)
- Must integrate with existing CAS (Central Authentication Service) for SSO
- Cannot introduce new runtime environments or languages without approval

**Rationale**: Organization has existing expertise, infrastructure, and support contracts for these technologies.

**Workaround**: None - this is a firm constraint based on organizational standards.

---

### CON-002: Browser Compatibility

**Type**: Technical  
**Impact**: High

The system must support legacy browser versions used within the organization.

**Constraints:**
- Must support HTML 3.2+ capable browsers at minimum
- Must function on Internet Explorer 11 (if still in use by organization)
- Must support modern browsers (Chrome, Firefox, Safari, Edge) - latest 2 versions
- Cannot require browser plugins or extensions
- Cannot use browser-specific proprietary features
- Must avoid JavaScript features not supported in older browsers (or use polyfills)

**Rationale**: Organization has mixed client environments; some departments use older hardware/software.

**Workaround**: Use transpilation (Babel) and polyfills; progressive enhancement strategy.

---

### CON-003: Existing Infrastructure

**Type**: Technical/Environmental  
**Impact**: High

The system must deploy on existing organizational infrastructure.

**Constraints:**
- Must run on existing servers (no new hardware budget for initial release)
- Must use existing load balancers and network infrastructure
- Must fit within current data center architecture
- Must use organization's SMTP server for email (no external email services)
- Must comply with existing firewall rules and network security policies
- Must use existing backup infrastructure

**Rationale**: Capital expenditure restrictions; must leverage existing investments.

**Workaround**: Optimize for efficiency; consider cloud migration in future phases if approved.

---

### CON-004: Database Limitations

**Type**: Technical  
**Impact**: Medium

The system must work within the constraints of the selected database system.

**Constraints:**
- Initial development may use lightweight database (Cloudscape/H2) for prototyping
- Production must use organization's licensed database (PostgreSQL/MySQL/Oracle)
- Cannot exceed database licensing limits (connection pools, storage)
- Must design for eventual migration to enterprise database if needed
- Cannot use database-specific proprietary features that limit portability

**Rationale**: Database licensing costs and organizational standards.

**Workaround**: Use ORM (Prisma/TypeORM) for database abstraction; design for portability.

---

### CON-005: Single Sign-On Integration

**Type**: Technical/Security  
**Impact**: High

The system must integrate with the existing portal's CAS-based authentication.

**Constraints:**
- Cannot implement independent authentication system
- Must use existing CAS server (version and configuration fixed)
- Must maintain session compatibility with portal framework
- Cannot modify CAS configuration or behavior
- Must accept CAS token format and claims as-is
- User credentials managed externally (no local user database for auth)

**Rationale**: Enterprise-wide SSO policy; security and user experience requirements.

**Workaround**: None - use Passport.js CAS strategy; adapt to CAS limitations.

---

### CON-006: Performance Requirements

**Type**: Technical  
**Impact**: Medium

The system must operate within defined performance parameters on existing hardware.

**Constraints:**
- Page load time must be ≤ 3 seconds on organization's network
- Must support 500 concurrent users on existing server capacity
- Database queries must complete within 500ms on current hardware
- Cannot require hardware upgrades for initial release
- Must operate within existing network bandwidth limitations

**Rationale**: Hardware refresh cycle limitations; budget constraints.

**Workaround**: Code optimization, caching strategies (Redis), efficient database queries.

---

## Business Constraints

### CON-100: Budget Limitations

**Type**: Business/Financial  
**Impact**: High

The project must be completed within allocated budget.

**Constraints:**
- No new hardware purchases for Release 1.0
- Limited budget for third-party libraries/services (prefer open-source)
- No budget for external consulting or additional full-time staff
- Must use existing software licenses
- Training budget limited to internal resources

**Rationale**: Fixed IT budget allocation; competing priorities.

**Workaround**: Leverage existing resources, open-source tools, and internal expertise.

---

### CON-101: Timeline Constraints

**Type**: Business/Project Management  
**Impact**: High

The project must meet fixed deployment deadlines.

**Constraints:**
- Release 1.0 must launch before next fiscal year (hard deadline)
- Cannot delay beyond Q4 2025 for business reasons
- Must align with annual HR policy updates cycle
- Parallel development with other organizational initiatives (shared resources)
- Limited testing window before production deployment

**Rationale**: Business cycle requirements; HR policy implementation deadlines.

**Workaround**: Prioritize critical features; defer nice-to-have features to Release 1.1/2.0.

---

### CON-102: Scope Limitations

**Type**: Business  
**Impact**: Medium

The system scope is limited to vacation time management only.

**Constraints:**
- Cannot include payroll integration (out of scope)
- Cannot include full HR management features (use existing HR systems)
- Cannot include time tracking for projects/billing (separate system)
- Cannot include benefits management (separate system)
- Must focus only on vacation requests, approvals, and basic grant management

**Rationale**: Project scope definition; avoid scope creep; other systems handle these functions.

**Workaround**: Provide integration points (APIs) for future extensions.

---

### CON-103: Change Management

**Type**: Business/Organizational  
**Impact**: Medium

System changes must follow established organizational change management procedures.

**Constraints:**
- All changes require approval from HR department
- Major changes require steering committee approval
- Cannot modify vacation policies without HR authorization
- Rule changes require business owner approval and documentation
- Production changes require change advisory board (CAB) approval

**Rationale**: Governance requirements; risk management; business ownership.

**Workaround**: Build flexible rules engine; delegate policy management to HR through UI.

---

## Organizational Constraints

### CON-200: Team Size and Composition

**Type**: Organizational/Resource  
**Impact**: High

Development team size and composition is fixed.

**Constraints:**
- Development team: 3-5 developers (NestJS/TypeScript experience varies)
- 1 part-time QA resource (shared with other projects)
- 1 system administrator (shared responsibility)
- No dedicated UI/UX designer (developer-designed interfaces)
- Product owner available 20% time (HR manager with other duties)

**Rationale**: Organizational staffing levels; resource allocation across multiple projects.

**Workaround**: Cross-training; knowledge sharing; iterative development with frequent feedback.

---

### CON-201: Development Environment

**Type**: Organizational/Technical  
**Impact**: Medium

Development must occur within organizational development environment constraints.

**Constraints:**
- Must use organization-approved IDEs (VS Code, WebStorm)
- Must use organization's Git repository (GitHub Enterprise/GitLab)
- Must use organization's CI/CD pipeline (Jenkins/GitLab CI)
- Cannot use external cloud development services without approval
- Must follow organization's code review process
- Must use organization's project management tools (Jira/Azure DevOps)

**Rationale**: Security policies; tooling standardization; license management.

**Workaround**: Adapt to existing tooling; automate where possible.

---

### CON-202: Skill Set Limitations

**Type**: Organizational/Resource  
**Impact**: Medium

Team has varying levels of expertise with chosen technologies.

**Constraints:**
- NestJS framework is new to some team members
- TypeScript experience varies across team
- Limited experience with modern frontend frameworks
- No dedicated DevOps engineer (learning required)
- Limited experience with microservices architecture

**Rationale**: Technology Node.js/NestJS stack.

**Workaround**: Training during development; pair programming; code reviews; documentation.

---

### CON-203: Access and Permissions

**Type**: Organizational/Security  
**Impact**: Medium

Development team has limited access to production systems and data.

**Constraints:**
- No direct access to production database
- No direct access to production servers
- Limited access to production logs (via system admin)
- Cannot test with real employee data in development (privacy concerns)
- Cannot access CAS configuration (managed by separate team)

**Rationale**: Security policies; separation of duties; data privacy regulations.

**Workaround**: Synthetic test data; staging environment; collaboration with operations team.

---

## Environmental Constraints

### CON-300: Network Environment

**Type**: Environmental/Technical  
**Impact**: Medium

The system must operate within organizational network constraints.

**Constraints:**
- Intranet-only deployment (no public internet access)
- Must operate through corporate proxy servers
- Firewall rules restrict outbound connections
- VPN required for remote access
- Network bandwidth shared with other applications
- Cannot open new firewall ports without security approval

**Rationale**: Corporate security policies; network architecture.

**Workaround**: Design for intranet deployment; coordinate with network team for required access.

---

### CON-301: Operating Environment

**Type**: Environmental/Technical  
**Impact**: Medium

The system must run on organization's server operating systems.

**Constraints:**
- Production servers: Linux (Ubuntu 24 or RHEL 8+) or Windows Server
- Cannot require OS features not available on these platforms
- Must use organization's approved container runtime (Docker)
- Must comply with OS hardening standards
- Limited sudo/admin privileges for deployment

**Rationale**: OS standardization; security hardening requirements.

**Workaround**: Docker containerization for consistency; follow OS best practices.

---

### CON-302: Deployment Windows

**Type**: Environmental/Operational  
**Impact**: Medium

System deployments must occur during approved maintenance windows.

**Constraints:**
- Deployments only during scheduled maintenance windows
- Production deployments: weekends or after 8 PM on weekdays
- Maximum 2-hour deployment window
- Emergency patches require executive approval
- Rollback must be possible within 30 minutes

**Rationale**: Business continuity; minimize disruption to users.

**Workaround**: Blue-green deployment; automated deployment scripts; thorough testing.

---

## Resource Constraints

### CON-400: Hardware Resources

**Type**: Resource/Technical  
**Impact**: High

System must operate within existing hardware resource limits.

**Constraints:**
- Server capacity: 4-8 CPU cores, 8-16 GB RAM
- Storage: 100 GB initial allocation (expandable with approval)
- Cannot require GPU or specialized hardware
- Must share server resources with other applications (initially)
- Network throughput: 1 Gbps shared

**Rationale**: Hardware refresh cycle not due; capital budget limitations.

**Workaround**: Efficient code; resource monitoring; horizontal scaling plan for future.

---

### CON-401: Licensing Constraints

**Type**: Resource/Legal  
**Impact**: Medium

Must use software with appropriate licensing for organizational use.

**Constraints:**
- Prefer open-source software (MIT, Apache 2.0 licenses)
- Commercial software requires legal and procurement approval
- Cannot exceed existing database connection limits
- Must comply with Node.js/npm package licenses
- Must track and approve all third-party dependencies

**Rationale**: Legal compliance; cost control; license management.

**Workaround**: Use permissive open-source licenses; license review process.

---

### CON-402: Support Resources

**Type**: Resource/Operational  
**Impact**: Medium

Limited support resources available post-deployment.

**Constraints:**
- 1 system administrator (part-time) for production support
- Development team provides Tier 2/3 support (limited availability)
- No 24/7 support (business hours only: 8 AM - 6 PM M-F)
- HR department provides Tier 1 support for business questions
- Limited on-call resources for emergencies

**Rationale**: Support team size; budget limitations.

**Workaround**: Comprehensive documentation; self-service features; monitoring/alerting.

---

## Legal and Regulatory Constraints

### CON-500: Data Privacy Regulations

**Type**: Legal/Compliance  
**Impact**: High

System must comply with applicable data privacy regulations.

**Constraints:**
- Must comply with GDPR (if applicable to organization)
- Must comply with CCPA (if applicable to California employees)
- Cannot store personal data outside approved jurisdictions
- Must provide data export capability for employee requests
- Must support data deletion/anonymization for former employees
- Audit logs must be tamper-proof and retained per regulations

**Rationale**: Legal requirements; regulatory compliance; potential fines for non-compliance.

**Workaround**: Privacy-by-design principles; data minimization; compliance review.

---

### CON-501: Employment Law Compliance

**Type**: Legal/Business  
**Impact**: High

Vacation management must comply with employment laws and company policies.

**Constraints:**
- Must enforce minimum vacation time requirements (per jurisdiction)
- Cannot violate collective bargaining agreements (if applicable)
- Must accommodate statutory holidays per location
- Must support different leave policies per jurisdiction
- Cannot prevent legally required time off requests
- Must maintain records per labor law requirements

**Rationale**: Legal obligations; risk of employment law violations.

**Workaround**: Flexible rules engine; location-based policy configuration.

---

### CON-502: Audit and Retention Requirements

**Type**: Legal/Compliance  
**Impact**: High

System must meet audit and record retention requirements.

**Constraints:**
- Activity logs retained for minimum 2 years (potentially up to 7 years)
- Vacation records retained for 3+ years per employment law
- Audit trails must be complete and tamper-evident
- Must support e-discovery requests
- Cannot delete records prematurely
- Backup retention must meet legal requirements

**Rationale**: Legal requirements; audit readiness; litigation preparedness.

**Workaround**: Automated archival process; immutable log storage; retention policy management.

---

### CON-503: Accessibility Requirements

**Type**: Legal/Compliance  
**Impact**: High

System must be accessible to users with disabilities.

**Constraints:**
- Must comply with WCAG 2.1 Level AA
- Must comply with Section 508 (if U.S. government contractor)
- Cannot require visual-only interaction
- Must support assistive technologies (screen readers)
- Must provide keyboard-only navigation
- Cannot discriminate against users with disabilities

**Rationale**: Legal requirements (ADA, Section 508); organizational inclusion policy.

**Workaround**: Accessibility testing; ARIA labels; semantic HTML; keyboard navigation.

---

## Integration Constraints

### CON-600: Legacy System Integration

**Type**: Integration/Technical  
**Impact**: High

Must integrate with existing HR legacy systems with limited flexibility.

**Constraints:**
- HR system API is read-only (cannot write back to legacy system)
- Data format is fixed (cannot request changes to legacy system)
- Integration runs on scheduled batch basis (not real-time)
- Legacy system maintained by external vendor (no direct control)
- Integration may fail; system must handle gracefully
- Data synchronization lag up to 24 hours acceptable

**Rationale**: Legacy system constraints; vendor limitations; cost of modifications.

**Workaround**: ETL process; data transformation layer; error handling; eventual consistency.

---

### CON-601: Portal Framework Constraints

**Type**: Integration/Technical  
**Impact**: High

Must operate within intranet portal framework limitations.

**Constraints:**
- Must use portal's look and feel (CSS framework provided)
- Must use portal's navigation structure
- Cannot modify portal framework code
- Limited to portal's supported integration patterns
- Must use portal's session management
- Portal framework version fixed (upgrades controlled by separate team)

**Rationale**: Enterprise portal managed by different team; consistency requirements.

**Workaround**: Adapt to portal framework; work within provided hooks and APIs.

---

### CON-602: Email System Constraints

**Type**: Integration/Technical  
**Impact**: Medium

Must work within corporate email system limitations.

**Constraints:**
- Must use corporate SMTP server (specific host and port)
- Email sending rate limited (anti-spam protection)
- Email templates must be plain text + HTML (no rich formats)
- Cannot guarantee email delivery (depends on recipient mail server)
- Attachment size limits (10 MB typical)
- Email content subject to corporate scanning/filtering

**Rationale**: Corporate email policies; security scanning; spam prevention.

**Workaround**: Email queue with retry logic; alternative in-app notifications.

---

## User Interface Constraints

### CON-700: Design Constraints

**Type**: UI/UX  
**Impact**: Medium

User interface must conform to organizational standards.

**Constraints:**
- Must use organization's design system/style guide
- Must maintain consistency with existing portal applications
- Limited to standard HTML/CSS (no heavy JavaScript frameworks for UI initially)
- No dedicated UI/UX designer available
- Must be usable without specialized training
- Must work on various screen sizes (desktop, tablet)

**Rationale**: Consistency with existing applications; ease of use requirement; resource constraints.

**Workaround**: Use portal's CSS framework; simple, clean design; user testing.

---

### CON-701: Input Method Constraints

**Type**: UI/Technical  
**Impact**: Low

User input must work across diverse client configurations.

**Constraints:**
- Cannot assume touch screen availability
- Cannot assume high-resolution displays
- Must work with keyboard and mouse only
- Cannot require specific browser plugins
- Cannot require JavaScript to be enabled for basic functionality (progressive enhancement)

**Rationale**: Diverse client hardware; accessibility; older workstations.

**Workaround**: Progressive enhancement; graceful degradation; responsive design.

---

## Constraints Summary Matrix

|ID|Constraint|Type|Impact|Workaround Possible|
|---|---|---|---|---|
|CON-001|Technology Stack Mandate|Technical|High|No|
|CON-002|Browser Compatibility|Technical|High|Partial|
|CON-003|Existing Infrastructure|Technical|High|Limited|
|CON-004|Database Limitations|Technical|Medium|Yes|
|CON-005|SSO Integration|Technical|High|No|
|CON-006|Performance Requirements|Technical|Medium|Yes|
|CON-100|Budget Limitations|Business|High|Limited|
|CON-101|Timeline Constraints|Business|High|No|
|CON-102|Scope Limitations|Business|Medium|Partial|
|CON-103|Change Management|Business|Medium|No|
|CON-200|Team Size|Organizational|High|Limited|
|CON-201|Development Environment|Organizational|Medium|Partial|
|CON-202|Skill Set Limitations|Organizational|Medium|Yes|
|CON-203|Access Permissions|Organizational|Medium|No|
|CON-300|Network Environment|Environmental|Medium|Limited|
|CON-301|Operating Environment|Environmental|Medium|Partial|
|CON-302|Deployment Windows|Environmental|Medium|Partial|
|CON-400|Hardware Resources|Resource|High|Limited|
|CON-401|Licensing Constraints|Resource|Medium|Yes|
|CON-402|Support Resources|Resource|Medium|Partial|
|CON-500|Data Privacy|Legal|High|No|
|CON-501|Employment Law|Legal|High|No|
|CON-502|Audit Requirements|Legal|High|No|
|CON-503|Accessibility|Legal|High|Partial|
|CON-600|Legacy Integration|Integration|High|Partial|
|CON-601|Portal Framework|Integration|High|Limited|
|CON-602|Email System|Integration|Medium|Yes|
|CON-700|Design Standards|UI/UX|Medium|Partial|
|CON-701|Input Methods|UI|Low|Yes|

---

