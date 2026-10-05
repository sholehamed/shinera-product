# Shinera — Development Status

**Project:** Shinera  
**Document Role:** Current implementation and delivery status  
**Last Updated:** 2026-10-05  
**Status Source:** Current project work + repository verification  
**Repository Sync:** Not yet synchronized with active local implementation

---

# 1. Purpose

This document answers one question:

> **Where is Shinera right now?**

It must describe the actual implementation state, not the desired roadmap.

Use:

- `SHINERA-PRD.md` for what the product should become.
- `SHINERA-PRODUCT-BACKLOG.md` for what should be built.
- `PROJECT-INSTRUCTIONS.md` for how work should be performed.
- `DEVELOPMENT-STATUS.md` for what is actually done now.

This document must be updated as implementation progresses.

---

# 2. Status Definitions

| Status | Meaning |
|---|---|
| `Done` | Implemented and considered complete for the current scope |
| `In Progress` | Active implementation is underway |
| `Partial` | Some important pieces exist, but the feature is not complete |
| `Needs Verification` | Prior work is known or expected, but repository confirmation is required |
| `Not Started` | No meaningful implementation is currently known |
| `Blocked` | Work cannot continue due to a known dependency or unresolved decision |

A feature must not be marked `Done` only because a UI mock, entity, endpoint stub, or partial implementation exists.

---

# 3. Repository Status

Shinera currently uses three GitHub repositories:

```text
sholehamed/shinera-product
sholehamed/shinera-backend
sholehamed/shinera-frontend
```

## Current GitHub State

As of 2026-10-05:

| Repository | GitHub State | Sync Status |
|---|---|---|
| `shinera-product` | README only | Product documents need to be committed |
| `shinera-backend` | README + gitignore only | Existing backend implementation not yet pushed |
| `shinera-frontend` | README + gitignore only | Existing frontend implementation not yet pushed |

### Important

The active Shinera implementation discussed and developed before creation of these repositories is not yet represented in the new GitHub repositories.

Therefore:

```text
Current local/project implementation != Current GitHub repository contents
```

Until the repositories are synchronized, implementation status below is based on the known current project state and must be revalidated after the initial push.

---

# 4. Overall Project Snapshot

Current high-level state:

```text
Product Definition        ██████████  Strong
Product Backlog           ██████████  Strong
Project Operating Rules   ██████████  Established

Backend Foundation        ████████░░  Mostly established
Frontend Foundation       █████████░  Established
Landing Page              ██████████  Complete

Authentication            █████░░░░░  Partial
Tenant / Branch           ██████░░░░  Partial
Subscription              ████░░░░░░  Partial / design underway
Registration              █████░░░░░  Partial
Dashboard UI              ███████░░░  Advanced UI, real data incomplete

Services                  ██░░░░░░░░  Early / needs verification
Staff                     ██░░░░░░░░  Early / needs verification
Customers                 █░░░░░░░░░  Not materially implemented
Appointments              ██░░░░░░░░  UI concepts/mocks exist; core domain pending
Payments                  █░░░░░░░░░  Not materially implemented
Public Booking            ░░░░░░░░░░  Not Started
Notifications             ░░░░░░░░░░  Not Started
```

The project is currently between **foundation/setup work** and the beginning of the **core business domain implementation**.

---

# 5. Product Documentation Status

| Artifact | Status | Notes |
|---|---|---|
| Product Vision | `Done` | Defined |
| PRD | `Done` | `SHINERA-PRD.md` prepared |
| MVP Scope | `Done` | Defined inside PRD |
| Product Backlog | `Done` | `SHINERA-PRODUCT-BACKLOG.md` prepared |
| Project Instructions | `Done` | `PROJECT-INSTRUCTIONS.md` prepared |
| Development Status | `In Progress` | This document |
| Definition of Done | `Not Started` | Next recommended document |
| Architecture Overview | `Not Started` | Should be extracted from current architecture |
| ADR Collection | `Not Started` | Decisions exist informally but are not yet recorded formally |

---

# 6. Backend Foundation Status

## SHN-E01 — Platform Foundation

| Story | Status | Current State |
|---|---|---|
| SHN-001 Solution Architecture | `Done` | .NET 10 solution structure established |
| SHN-002 Result Pattern | `Partial` | `Result<T>` / API response patterns have been actively implemented and refined |
| SHN-003 Global Exception Handling | `Done` | Global exception handling and sample error path implemented |
| SHN-004 Validation Pipeline | `Needs Verification` | Current repository must confirm final validator/pipeline convention |
| SHN-005 Database Infrastructure | `Done` | EF Core, database setup/migrations and DB health flow established |
| SHN-006 Audit Fields | `Needs Verification` | Architecture intent exists; actual implementation must be verified |
| SHN-007 Soft Delete | `Needs Verification` | Planned architecture, implementation not currently confirmed |

### Known Foundation Work

The following backend foundation work has already been completed or demonstrated in the active project:

- .NET 10 solution skeleton
- Domain project
- Application project
- Infrastructure project
- API project
- Application tests project
- Integration tests project
- Scalar API documentation
- Swagger removed in favor of Scalar
- Health endpoints
- Database create/migrate flow
- Global exception handling
- CQRS-oriented command/query implementation
- Minimal API endpoints
- `IDispatcher`-based dispatching
- Result/API response work

### Verification Needed After GitHub Sync

After initial backend push, verify:

```text
Solution/project dependencies
Current package versions
Migration history
Result types
Global error format
Health endpoints
Test project configuration
Dispatcher abstractions
```

---

# 7. Authentication & Identity Status

## SHN-E02 — Authentication & Identity

| Story | Status | Notes |
|---|---|---|
| SHN-010 User Entity | `Needs Verification` | User model exists conceptually but current implementation must be inspected |
| SHN-011 OpenIddict Integration | `Partial` | OpenIddict selected and integration work has started |
| SHN-012 Login | `Partial` | Frontend login exists; backend completion needs verification |
| SHN-013 Refresh Token | `Needs Verification` | Required architecture known; final implementation not confirmed |
| SHN-014 Logout | `Needs Verification` | Not confirmed end-to-end |
| SHN-015 Current User | `Not Started / Needs Verification` | No confirmed complete `/auth/me` flow |

### Known Work

OpenIddict is the selected authentication server.

Previous implementation work encountered and addressed EF/OpenIddict mapping concerns, including custom application model configuration.

The final authentication flow must still be validated as a complete chain:

```text
Login
→ Access Token
→ Refresh Token
→ Refresh
→ Revocation
→ Logout
→ Current User
```

---

# 8. Tenant & Workspace Status

## SHN-E03 — Tenant & Workspace

| Story | Status | Notes |
|---|---|---|
| SHN-020 Tenant Entity | `Partial` | Tenant entity/configuration work exists |
| SHN-021 Create Tenant | `Partial` | Tenant commands and domain work exist |
| SHN-022 Update Tenant | `Partial` | `UpdateTenantCommand` and handler have been worked on |
| SHN-023 Tenant Status | `Partial` | `SetTenantStatusCommand` flow has been implemented/discussed |
| SHN-024 Current Tenant Context | `Needs Verification` | Core design exists; final implementation needs repository verification |
| SHN-025 Tenant Query Isolation | `Needs Verification` | Critical test coverage must be confirmed |

### Known Implemented Work

Backend work includes examples such as:

```text
UpdateTenantCommand
SetTenantStatusCommand
Tenant list query
Tenant details query
```

Typed API response/result mapping has also been refined around these flows.

### Critical Remaining Requirement

Tenant isolation must not be considered complete until integration tests prove:

```text
Tenant A cannot read Tenant B
Tenant A cannot update Tenant B
Tenant A cannot reference Tenant B resources
```

---

# 9. Branch Status

## SHN-E04 — Branch Management

| Story | Status | Notes |
|---|---|---|
| SHN-030 Branch Entity | `Partial` | Branch domain/configuration work exists |
| SHN-031 Main Branch | `Partial` | Main Branch rule is defined |
| SHN-032 Branch CRUD | `Needs Verification` | Full endpoint set not yet confirmed |
| SHN-033 Set Main Branch | `Partial` | Command/handler work exists |
| SHN-034 Current Branch | `Not Started / Needs Verification` | Workspace branch switching not confirmed |

Known application work includes:

```text
SetMainBranchCommand
SetMainBranchHandler
```

The final invariant that each Tenant has exactly one valid Main Branch still requires database/application verification.

---

# 10. Registration Status

## SHN-E05 — Registration

| Story | Status | Notes |
|---|---|---|
| SHN-040 Registration Wizard | `In Progress` | Frontend wizard design/implementation underway |
| SHN-041 Plan Selection | `Partial` | Query-param plan flow designed and worked on |
| SHN-042 Business Registration | `Partial` | UI/data requirements defined |
| SHN-043 Owner Registration | `Partial` | UI/data requirements defined |
| SHN-044 Registration Transaction | `Not Started / Needs Verification` | Full atomic backend workflow not confirmed |
| SHN-045 Registration Success | `Not Started` | Full login → onboarding chain not complete |

### Current Registration Design

The wizard collects:

```text
Tenant
Business Profile
Branch
Owner User
Selected Plan
```

The expected backend transaction is:

```text
Create User
Create Tenant
Create Business Profile
Create Main Branch
Create Membership
Create Subscription
Assign Owner
```

This flow should not be marked complete until it is implemented transactionally and covered by integration tests.

---

# 11. Business Profile Status

## SHN-E06 — Business Profile

| Story | Status | Notes |
|---|---|---|
| SHN-050 Business Profile Entity | `Needs Verification` | Model requirements exist |
| SHN-051 Business Profile Update | `Not Started / Needs Verification` | No confirmed completed feature |
| SHN-052 Business Logo | `Not Started` | Post-foundation task |

---

# 12. Plans, Features & Subscription Status

## SHN-E07 — Plans & Subscription

| Story | Status | Notes |
|---|---|---|
| SHN-060 Plan Entity | `Partial` | Entity/configuration work requested/started |
| SHN-061 Feature Entity | `Partial` | Entity/configuration work requested/started |
| SHN-062 Plan Feature | `Needs Verification` | Relationship design exists |
| SHN-063 Subscription | `Partial / Needs Verification` | Registration dependency established |
| SHN-064 Feature Gating | `Not Started / Needs Verification` | Final backend enforcement not confirmed |

### Product Decision Already Established

Feature access should be capability-based.

Preferred:

```text
Feature.MultipleBranches
Feature.AdvancedReports
```

Avoid scattered checks such as:

```text
plan == "SalonPro"
```

---

# 13. Authorization Status

## SHN-E08 — Authorization

| Story | Status | Notes |
|---|---|---|
| SHN-070 Permission Catalog | `Design Established` | Dynamic resource/action model defined |
| SHN-071 Roles | `Design Established` | Owner/Admin/Receptionist/Staff baseline |
| SHN-072 Permission Scope | `Design Established` | Tenant/Branch/Own/Child model designed |
| SHN-073 Endpoint Authorization | `Not Started / Needs Verification` | Full enforcement not confirmed |
| SHN-074 Frontend Permission Integration | `Not Started` | Not confirmed |

### Known Architecture

Authorization is intended to be modular and dynamic.

Core concepts already selected:

```text
ApiResource
UiResource
Permission
PermissionAssignment

Subject:
User / Role

Scope:
Tenant / Branch / Own / Child
```

UI resource and API resource catalogs are intentionally separate.

This is a major architectural subsystem and should receive its own ADR before large-scale implementation.

---

# 14. Landing Page Status

**Frontend Phase 02**

Status:

```text
Done
```

Known completed sections:

```text
Banner / Hero
Key Features
Widgets
Testimonials
FAQ
Contact
CTA
Footer
```

Known product/UI characteristics:

- Persian content
- RTL
- Angular Material / Trezo
- Shinera dusty-rose brand palette
- Dark/light theme compatibility
- Responsive landing layout

The landing page is considered complete for the current phase, subject to future content/polish changes.

---

# 15. Frontend Authentication Status

## Login

Status:

```text
Partial / UI Advanced
```

Known work:

- Login-only layout
- Removed unnecessary product marketing content from login
- Customer-compatible login concept
- Dark/light mode styling
- Captcha component integration
- Desktop height/overflow improvements
- Input visibility fixes in dark mode

Remaining:

- Verify final API integration
- Verify token lifecycle
- Verify refresh flow
- Verify error states
- Verify authorization redirect behavior
- Add E2E coverage

---

# 16. Dashboard Status

Status:

```text
Partial — UI implementation significantly ahead of backend data integration
```

Known dashboard components include:

### Recent Appointments

Implemented UI concepts:

- Persian date formatting
- date selection
- appointment status presentation
- client/staff information
- empty state
- fixed-height behavior
- RTL/dark mode integration
- mock appointment data

### Revenue By Services

Implemented UI/chart concepts:

- Daily
- Weekly
- Monthly
- ApexCharts stacked bar presentation

### Featured Services

Implemented UI concepts:

- service ranking
- served count
- revenue
- trend
- performance percentage

### Other KPI / Widget Work

Known UI work includes:

- Customer acquisition channels
- New Customers
- Customer Satisfaction
- Total Orders / revenue-oriented KPI cards
- Welcome/dashboard visual components

### Remaining Before Dashboard Is Product-Complete

```text
Replace mock data with backend queries
Define dashboard API contract
Branch filtering
Date range filtering
Loading states
Error states
Permission handling
Feature gating
Real revenue aggregation
Real appointment aggregation
Performance validation
```

---

# 17. Services Status

## SHN-E09 / SHN-E10

Overall:

```text
Needs Verification / Early
```

Product model and backend entity/configuration requirements are defined.

A complete production-ready flow is not currently confirmed:

```text
Category CRUD
Service CRUD
Activation
Filtering
Permissions
Tenant isolation
Frontend integration
Tests
```

Do not mark Services `Done` until this full vertical slice exists.

---

# 18. Staff & Schedule Status

## SHN-E11 / SHN-E12

Overall:

```text
Not Started / Needs Verification
```

The product rules are defined:

```text
Staff
StaffBranch
StaffService
Weekly Schedule
Breaks
Days Off
Special Schedule
Time Off
```

No complete end-to-end implementation is currently confirmed.

---

# 19. Customers Status

## SHN-E13

Overall:

```text
Not Started
```

Required capabilities are defined but no complete implementation is currently known:

```text
Customer CRUD
Search
Duplicate detection
Notes
Appointment history
Payment history
Tenant isolation
```

---

# 20. Appointment Status

## SHN-E14 / SHN-E15

Overall:

```text
Not Started for core domain
Partial for dashboard/mock UI
```

Important distinction:

The dashboard currently contains appointment-oriented UI/mock data.

This does **not** mean the Appointment domain is implemented.

Core appointment work still includes:

```text
Appointment entity
State machine
Availability engine
Schedule evaluation
Conflict detection
Concurrency protection
Create appointment
Reschedule
Cancel
Start
Complete
NoShow
Queries
Calendar API
Integration tests
E2E
```

This is the highest-value remaining core domain.

---

# 21. Payments Status

## SHN-E16

Overall:

```text
Not Started
```

Product requirements exist, but no complete implementation is currently confirmed.

---

# 22. Onboarding Status

## SHN-E18

Overall:

```text
Not Started / Product flow defined
```

Expected checklist:

```text
Business Profile
First Service
First Staff
Working Hours
First Customer
First Appointment
```

---

# 23. Public Booking Status

## SHN-E19

Status:

```text
Not Started
```

This remains a post-core-MVP / launch feature.

---

# 24. Notifications Status

## SHN-E20

Status:

```text
Not Started
```

Infrastructure should remain minimal until appointment flows are stable.

---

# 25. Audit Trail Status

## SHN-E21

Status:

```text
Needs Verification
```

Audit requirements are known, but implementation depth must be confirmed after repository sync.

---

# 26. Localization Status

## SHN-E22

| Area | Status |
|---|---|
| Persian UI | `Done / Established` |
| RTL foundation | `Done / Established` |
| fa-IR display | `Partial` |
| Persian calendar UI | `Partial` |
| Gregorian persistence rule | `Design Established` |
| UTC strategy | `Design Established / Needs Verification` |
| Money formatting abstraction | `Not Started / Needs Verification` |

---

# 27. Search & Pagination Status

## SHN-E23

Status:

```text
Needs Verification
```

List endpoint conventions are defined but must be standardized after repository sync.

---

# 28. Observability Status

## SHN-E24

Current state:

```text
Health Checks: Done
Structured Logging: Needs Verification
Tracing: Needs Verification
Exception Monitoring: Partial
Application Monitoring Dashboard: Separate package exploration exists
```

The reusable AppMonitoring package discussion is separate from Shinera MVP and should not block core product delivery.

---

# 29. Security Status

## SHN-E25

Current status:

```text
Partial / Architecture Defined
```

Important security requirements are identified:

- Tenant isolation
- Permission enforcement
- OpenIddict token lifecycle
- Rate limiting
- Public booking protection
- Branch access
- secure error responses

Security should be validated vertically per feature rather than postponed until the end.

---

# 30. Testing Status

## SHN-E26

Overall:

```text
Partial
```

Known solution structure includes:

```text
Application.Tests
IntegrationTests
```

Current coverage depth must be verified after repository sync.

Critical suites still required:

```text
Authentication
Tenant isolation
Authorization
Service CRUD
Staff assignment
Customer CRUD
Appointment availability
Appointment conflict
Appointment state transitions
Payments
Registration
```

Playwright E2E strategy is defined but full critical-path coverage is not yet complete.

---

# 31. CI/CD Status

## SHN-E27

Status:

```text
Not Started / Needs Verification
```

Target pipeline:

```text
Restore
Build
Unit Tests
Integration Tests
Frontend Build
Playwright
Publish
```

---

# 32. Current MVP Completion Matrix

| Area | Status | MVP Criticality |
|---|---|---|
| Product Requirements | `Done` | P0 |
| Backlog | `Done` | P0 |
| Project Instructions | `Done` | P0 |
| Backend Foundation | `Mostly Done` | P0 |
| Frontend Foundation | `Done` | P0 |
| Landing Page | `Done` | Pre-MVP |
| Authentication | `Partial` | P0 |
| Tenant | `Partial` | P0 |
| Branch | `Partial` | P0 |
| Permissions | `Design / Partial` | P0 |
| Subscription | `Partial` | P0 |
| Registration | `In Progress` | P0 |
| Business Profile | `Needs Verification` | P0 |
| Services | `Early / Needs Verification` | P0 |
| Staff | `Not Started / Needs Verification` | P0 |
| Schedule | `Not Started` | P0 |
| Customers | `Not Started` | P0 |
| Appointments | `Core Not Started` | P0 |
| Payments | `Not Started` | P0 |
| Dashboard UI | `Partial` | P0 |
| Dashboard Real Data | `Not Started` | P0 |
| Onboarding | `Not Started` | P1 |
| Public Booking | `Not Started` | P1 |
| Notifications | `Not Started` | P1 |
| CI/CD | `Needs Verification` | P1 |

---

# 33. Current Phase Assessment

Shinera is currently best described as:

```text
Foundation established
+
Product definition established
+
Frontend visual direction established
+
Tenant/subscription/auth groundwork underway
+
Core salon business flows not yet implemented end-to-end
```

The biggest mistake at this stage would be continuing to add dashboard polish while delaying the core transactional domain.

The next development focus should move toward completing platform prerequisites and then entering:

```text
Service
→ Staff
→ Schedule
→ Customer
→ Appointment
```

---

# 34. Immediate Repository Milestone

Before treating GitHub as the engineering source of truth:

## Backend

Push the current active backend project into:

```text
sholehamed/shinera-backend
```

Then verify:

```text
Solution structure
Build
Migrations
Tests
Scalar
Health
Tenant features
OpenIddict work
Result pattern
```

## Frontend

Push the current active Angular project into:

```text
sholehamed/shinera-frontend
```

Then verify:

```text
Angular version
Build
Landing
Login
Registration
Dashboard
Theme
RTL
Current API integration
```

## Product

Commit:

```text
SHINERA-PRD.md
SHINERA-PRODUCT-BACKLOG.md
PROJECT-INSTRUCTIONS.md
DEVELOPMENT-STATUS.md
```

to:

```text
shinera-product
```

---

# 35. Recommended Next Development Sequence

After repository synchronization:

```text
1. Repository baseline verification
2. Backend build + tests
3. Frontend build
4. Reconcile DEVELOPMENT-STATUS.md with actual code
5. Finish authentication lifecycle
6. Finish CurrentTenant + tenant isolation tests
7. Finish Branch core
8. Finish Plan / Feature / Subscription
9. Complete registration transaction
10. Implement Business Profile
11. Implement Services vertical slice
12. Implement Staff vertical slice
13. Implement Schedule engine
14. Implement Customer vertical slice
15. Start Appointment domain
```

---

# 36. Next Core Milestone

The next meaningful product milestone should be:

## M1 — Workspace Ready

A newly registered Owner can:

```text
Choose Plan
→ Register
→ Create Tenant
→ Create Main Branch
→ Receive Subscription
→ Login
→ Enter Workspace
```

Required areas:

```text
Authentication
Tenant
Branch
Plan
Feature
Subscription
Registration
Owner membership
Permissions
```

Only after M1 is stable should the main business setup milestone become the primary focus.

---

# 37. Following Milestone

## M2 — Business Ready

Owner can:

```text
Configure Business
→ Create Service
→ Create Staff
→ Assign Staff to Service
→ Configure Staff Schedule
```

---

# 38. Core MVP Milestone

## M3 — Booking Ready

Owner / Receptionist can:

```text
Create Customer
→ Select Service
→ Select Staff
→ Calculate Available Slots
→ Create Appointment
→ Prevent Conflict
```

This is the point where Shinera begins delivering its core product value.

---

# 39. Status Update Rules

This document should be updated when:

- a backlog story starts
- a story reaches backend completion
- a story reaches frontend completion
- integration is completed
- tests pass
- a feature is blocked
- scope changes
- a major repository sync occurs

Do not wait until the end of a sprint to update major status changes.

---

# 40. Story Status Format

When implementation starts, use entries such as:

```text
SHN-132 — Create Appointment
Status: In Progress
Backend: In Progress
Frontend: Not Started
Database: Done
Authorization: Partial
Tests: In Progress
Blocked By: SHN-133 Available Slots
Last Updated: YYYY-MM-DD
```

When completed:

```text
SHN-132 — Create Appointment
Status: Done
Backend: Done
Frontend: Done
Database: Done
Authorization: Done
Tests: Done
Completed: YYYY-MM-DD
```

---

# 41. Verification Principle

Whenever this document conflicts with the repositories:

```text
Repository reality wins for implementation status.
```

Then this document must be updated.

Whenever repository behavior conflicts with an approved product requirement:

```text
Do not change the requirement silently.
```

The conflict must be reviewed as product/architecture work.

---

# 42. Current Summary

As of 2026-10-05:

```text
Shinera has a strong product definition and substantial foundation/UI work.

The project is not yet at core MVP completion.

The immediate operational priority is:
1. synchronize the new repositories,
2. verify the real code baseline,
3. complete workspace/auth/tenant/subscription foundations,
4. move quickly into Service → Staff → Schedule → Customer → Appointment.
```
