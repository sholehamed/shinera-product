# Shinera — Architecture Overview

**Project:** Shinera  
**Document Role:** High-level technical architecture reference  
**Audience:** Developers, architects, reviewers, ChatGPT, Work, coding agents  
**Status:** Active  
**Version:** 1.0

---

# 1. Purpose

This document describes the high-level architecture of Shinera.

It answers:

- What are the main architectural building blocks?
- How are backend, frontend, product documentation, and data separated?
- What are the main domain boundaries?
- How does multi-tenancy work?
- How do authentication, authorization, subscription, and feature gating fit together?
- How should data and requests flow through the system?
- Which architectural decisions are already established?
- Which decisions still require ADRs?

This document is intentionally high-level.

Detailed decisions that have long-term architectural impact should be documented separately as ADRs.

---

# 2. System Context

Shinera is a multi-tenant SaaS for beauty businesses and solo professionals.

Primary users:

```text
Owner
Admin
Receptionist
Staff / Specialist
Customer
```

Primary product capabilities:

```text
Authentication
Registration
Tenant / Workspace
Branch
Subscription
Permissions
Business Profile
Services
Staff
Schedules
Customers
Appointments
Payments
Dashboard
Public Booking
Notifications
```

---

# 3. Repository Architecture

Shinera is intentionally split into three repositories.

```text
shinera-product
shinera-backend
shinera-frontend
```

## shinera-product

Contains:

```text
PRD
Product Backlog
Project Instructions
Development Status
Definition of Done
Architecture Overview
ADRs
Roadmap / product docs
```

Purpose:

```text
What should be built?
Why should it be built?
What decisions govern the project?
```

## shinera-backend

Contains:

```text
Domain
Application
Infrastructure
API
Tests
Migrations
```

Purpose:

```text
Business rules
Persistence
Security
API
Backend integration
```

## shinera-frontend

Contains:

```text
Angular application
UI
Routes
Components
Forms
API integration
Theme
RTL
Playwright
```

Purpose:

```text
User experience
Client-side workflows
Presentation
```

---

# 4. Architectural Style

Backend architecture:

```text
Modular Monolith
+
Vertical Slice Architecture
+
CQRS
+
Minimal API
```

This combination is chosen to provide:

- strong separation of concerns,
- feature-oriented organization,
- easier testing,
- lower accidental complexity than microservices,
- room for future modular extraction if justified.

Shinera is not designed as a distributed microservice system in the MVP.

---

# 5. Backend Solution Structure

Current intended structure:

```text
Shinera
│
├── Domain
├── Application
├── Infrastructure
├── Api
├── Application.Tests
└── IntegrationTests
```

## Domain

Responsible for:

```text
Entities
Value Objects
Business invariants
Domain behavior
State transitions
```

Domain must not depend on:

```text
Infrastructure
API
EF Core-specific behavior where avoidable
External services
```

## Application

Responsible for:

```text
Commands
Queries
Handlers
DTOs
Validation
Application orchestration
Interfaces / abstractions
Result handling
```

Application depends on Domain.

## Infrastructure

Responsible for:

```text
EF Core
Database
OpenIddict persistence/integration
External service implementations
File storage
Notification providers
Infrastructure adapters
```

Infrastructure implements Application abstractions.

## API

Responsible for:

```text
Composition Root
Minimal API endpoints
Authentication middleware
Authorization middleware/policies
Request/response mapping
OpenAPI / Scalar exposure
Health endpoints
```

API should remain thin.

---

# 6. Dependency Direction

Expected dependency direction:

```text
API
 ↓
Application
 ↓
Domain

Infrastructure
 ↓
Application
 ↓
Domain
```

Conceptually:

```text
Domain
   ↑
Application
   ↑
API

Infrastructure → Application
```

Domain should not know about the outer layers.

---

# 7. Vertical Slice Architecture

Features should be organized around business capability rather than technical layer where practical.

Example:

```text
Features/
└── Tenancy/
    └── Tenants/
        ├── CreateTenantCommand
        ├── CreateTenantHandler
        ├── GetTenantQuery
        ├── GetTenantHandler
        ├── TenantDto
        └── Validation
```

Another example:

```text
Features/
└── Appointments/
    ├── CreateAppointment
    ├── RescheduleAppointment
    ├── CancelAppointment
    ├── GetAvailableSlots
    └── GetAppointments
```

Exact folder names should follow the existing repository conventions.

The architecture principle is more important than the folder spelling.

---

# 8. CQRS

Shinera separates reads and writes.

## Commands

Commands mutate state.

Examples:

```text
CreateTenantCommand
UpdateTenantCommand
SetMainBranchCommand
SetTenantStatusCommand
CreateAppointmentCommand
CancelAppointmentCommand
```

## Queries

Queries return data.

Examples:

```text
GetTenantsQuery
GetTenantQuery
GetAvailableSlotsQuery
GetAppointmentsQuery
GetDashboardQuery
```

## Dispatcher

Shinera uses its own/current dispatcher abstraction:

```text
IDispatcher
```

Do not replace it with MediatR merely for convention.

---

# 9. Result Pattern

Expected business failures should use:

```text
Result
Result<T>
```

API responses use the project's response envelope:

```text
ApiResponse
ApiResponse<T>
```

Typical flow:

```text
Handler
  ↓
Result<T>
  ↓
Endpoint
  ↓
TypedResult / ApiResponse<T>
```

Expected errors include:

```text
validation failure
not found
business rule violation
conflict
feature unavailable
access denied
```

Exceptions are reserved for unexpected or infrastructure-level failures unless explicitly modeled otherwise.

---

# 10. API Architecture

Shinera uses:

```text
ASP.NET Core Minimal API
```

API documentation:

```text
Scalar
```

Swagger UI is not part of the current preferred setup.

Typical endpoint responsibilities:

```text
Receive Request
↓
Bind parameters
↓
Authorize
↓
Dispatch Command / Query
↓
Map Result
↓
Return typed response
```

Endpoints must not become business-service classes.

---

# 11. Data Access

Primary ORM:

```text
EF Core
```

Primary abstraction:

```text
IApplicationDbContext
```

Commands and queries may access the abstraction directly according to the current project style.

Use:

- projection for read models,
- server-side filtering,
- server-side pagination,
- explicit includes only when required.

Avoid:

```text
N+1 queries
unbounded list loading
unnecessary entity materialization
automatic CompiledQuery usage everywhere
```

Optimization must be based on evidence.

---

# 12. Persistence Strategy

Primary relational database:

```text
Relational DB via EF Core
```

Exact production provider should follow repository/environment configuration.

Persistence concerns include:

```text
Entity configurations
Indexes
Constraints
Migrations
Transactions
Concurrency
Audit fields
Soft delete where appropriate
```

Business-critical consistency should be protected by both:

```text
Application/domain validation
+
Database integrity where appropriate
```

---

# 13. Multi-Tenancy Architecture

Multi-tenancy is a first-class architecture concern.

Each business is represented by:

```text
Tenant
```

Core hierarchy:

```text
Tenant
├── BusinessProfile
├── Branches
├── Users / Memberships
├── Staff
├── Services
├── Customers
├── Appointments
├── Payments
└── Subscription
```

Tenant isolation is a security boundary.

---

# 14. System Tenant

A System Tenant may exist for platform-level/system-owned data where required by installation or platform logic.

System Tenant behavior must not weaken normal Tenant isolation.

Platform-level data must be clearly distinguished from tenant-owned data.

---

# 15. Current Tenant Context

Application code should rely on a current-context abstraction such as:

```text
ICurrentTenant
```

Conceptual data:

```text
TenantId
BranchId?
UserId
```

Rules:

- Do not trust client-provided `TenantId` as authorization.
- Resolve tenant context from authenticated/session context.
- Validate related entity ownership.
- Prevent cross-tenant object reference attacks.

---

# 16. Tenant Membership

Users may belong to a Tenant through membership.

Conceptually:

```text
User
  ↓
TenantMembership
  ↓
Tenant
```

Membership may define:

```text
role
status
access scope
```

The final model must align with the permission architecture.

---

# 17. Branch Architecture

Tenant may have one or more Branches.

```text
Tenant
 ├── Main Branch
 ├── Branch 2
 └── Branch N
```

Rules:

- Main Branch is created during workspace provisioning.
- A Tenant must have a valid Main Branch.
- Multi-branch availability may depend on Subscription Feature.
- Branch access is not automatically granted merely by Tenant membership.

Staff may belong to multiple branches.

---

# 18. Branch Membership

Conceptual model:

```text
User / Staff
   ↓
BranchMembership
   ↓
Branch
```

Branch membership and role/permission scope are related but should not be conflated.

Membership answers:

```text
Where can this subject operate?
```

Permission answers:

```text
What can this subject do?
```

---

# 19. Authentication Architecture

Authentication is based on:

```text
OpenIddict
```

ASP.NET Identity is not the preferred core identity architecture.

Expected authentication capabilities:

```text
Login
Access Token
Refresh Token
Token Rotation
Token Revocation
Logout
Current User
```

OpenIddict persistence must integrate cleanly with the application's EF Core setup.

---

# 20. User Model

Shinera owns its business User model.

Conceptual fields:

```text
Id
FirstName
LastName
Phone
Email
PasswordHash
IsActive
LastLoginAt
```

Authentication protocol concerns and Shinera business identity should remain conceptually separated even if they are persisted in related data stores.

---

# 21. Authorization Architecture

Shinera uses permission-oriented authorization rather than simple role checks.

Core concepts:

```text
ApiResource
UiResource
Permission
PermissionAssignment
```

Subjects:

```text
User
Role
```

Scopes:

```text
Tenant
Branch
Own
Child
```

Only scopes actually implemented should be used in production behavior.

---

# 22. Permission Model

Conceptually:

```text
Subject
  ↓
PermissionAssignment
  ↓
Permission
  ↓
Resource + Action
```

Examples:

```text
appointments.view
appointments.create
appointments.update
appointments.cancel

customers.view
customers.create

staff.manage
services.manage
```

The final runtime permission model should support dynamic authorization without requiring hardcoded role checks across endpoints.

---

# 23. Role Model

Initial roles:

```text
Owner
Admin
Receptionist
Staff
```

Roles are permission containers.

Avoid logic like:

```text
if role == Owner
```

for behavior that should actually be represented by a permission.

Role-specific business behavior may exist only when it is genuinely a role concept.

---

# 24. API Resource vs UI Resource

API and UI catalogs are intentionally separated.

Example:

```text
ApiResource:
appointments

UiResource:
appointments.page
appointments.create.button
```

The backend API catalog exists for authorization.

The UI resource catalog exists for experience control.

UI resource visibility is not a security boundary.

---

# 25. Subscription Architecture

Tenant owns a Subscription.

```text
Tenant
  ↓
Subscription
  ↓
Plan
  ↓
PlanFeature
  ↓
Feature
```

Core entities:

```text
Plan
Feature
PlanFeature
Subscription
```

Initial plans:

```text
Solo
Solo Pro
Salon
Salon Pro
```

---

# 26. Feature Gating

Feature checks should use capabilities.

Preferred:

```text
Feature.MultipleBranches
Feature.AdvancedReports
Feature.ForceAppointment
```

Avoid distributed plan-name logic:

```text
if plan == "SalonPro"
```

unless explicitly required.

Feature gating exists in two layers:

```text
Frontend
→ UX visibility / upgrade messaging

Backend
→ real entitlement enforcement
```

Backend is authoritative.

---

# 27. Registration / Provisioning Architecture

Registration is not merely User creation.

It provisions a workspace.

Conceptual transaction:

```text
Create User
↓
Create Tenant
↓
Create BusinessProfile
↓
Create Main Branch
↓
Create TenantMembership
↓
Create Subscription
↓
Assign Owner permissions/role
```

This flow should be atomic.

If provisioning fails, partial workspace creation should not remain silently.

---

# 28. Workspace Context

After login:

```text
User
 ↓
Tenant
 ↓
Branch
 ↓
Application
```

If there is only one valid Tenant:

```text
Tenant selection may be skipped
```

If there is only one valid Branch:

```text
Branch selection may be skipped
```

The active workspace context influences:

```text
Queries
Permissions
Branch filtering
Dashboard data
Appointment operations
```

---

# 29. Business Profile

BusinessProfile contains tenant-facing business identity.

Conceptually:

```text
Tenant
  ↓
BusinessProfile
```

Possible data:

```text
DisplayName
BusinessType
Phone
Email
Address
Logo
Description
WorkingHours
```

Tenant identity and public business presentation should not be unnecessarily coupled.

---

# 30. Service Domain

Conceptual model:

```text
ServiceCategory
   ↓
Service
```

Service:

```text
Name
Duration
Price
Description
Active Status
```

Future extensions may include:

```text
Branch-specific pricing
Staff-specific pricing
Add-ons
Packages
Discounts
```

These are not part of the core MVP unless separately approved.

---

# 31. Staff Domain

Conceptual model:

```text
Staff
├── StaffBranch
├── StaffService
└── StaffSchedule
```

Important distinction:

```text
Staff != User
```

A Staff member may exist without a login account.

A Staff member may optionally reference a User.

This allows salons to manage specialists who do not use the system directly.

---

# 32. Schedule Domain

Schedule is a core dependency of Appointment availability.

Conceptual components:

```text
WeeklySchedule
Break
DayOff
SpecialSchedule
TimeOff
```

Schedule evaluation should produce an effective availability window for a staff member on a specific date.

Avoid duplicating schedule logic inside multiple appointment endpoints.

---

# 33. Customer Domain

Customer belongs to Tenant.

Conceptually:

```text
Customer
├── Appointments
├── Payments
├── Notes
└── Activity
```

Customer identity may initially be matched by:

```text
Mobile
```

Duplicate detection should be tenant-aware.

A phone number may legitimately exist as a customer in multiple Tenants.

---

# 34. Appointment Domain

Appointment is the core transactional domain of Shinera.

Conceptual relationship:

```text
Appointment
├── Tenant
├── Branch
├── Customer
├── Staff
├── Service
├── Date / Time
├── Price
├── Status
└── Notes
```

The Appointment domain coordinates multiple business rules.

---

# 35. Appointment State Model

Current conceptual states:

```text
Pending
Confirmed
Upcoming
InProgress
Completed
Cancelled
NoShow
```

Possible transitions:

```text
Pending
  ↓
Confirmed
  ↓
InProgress
  ↓
Completed
```

Alternatives:

```text
Pending → Cancelled
Confirmed → Cancelled
Confirmed → NoShow
```

Invalid transitions must be rejected by backend/domain logic.

Exact state semantics should be finalized in an ADR or dedicated domain design before full implementation.

---

# 36. Appointment Availability

Availability depends on:

```text
Tenant
Branch
Service
Staff
Service duration
Staff schedule
Breaks
Special schedule
Time off
Existing appointments
Appointment status
```

Conceptual engine:

```text
Requested Date
   ↓
Resolve Staff Schedule
   ↓
Apply Exceptions
   ↓
Subtract Breaks
   ↓
Subtract Existing Appointments
   ↓
Apply Service Duration
   ↓
Generate Available Slots
```

Public Booking and internal booking must reuse the same availability engine.

---

# 37. Appointment Conflict

Conflict detection must occur in Backend.

Frontend-generated availability is advisory.

Final validation:

```text
Request arrives
↓
Re-check availability
↓
Protect against concurrency
↓
Create appointment
```

The exact concurrency strategy must be documented separately.

Recommended ADR:

```text
ADR-004 Appointment Concurrency Strategy
```

---

# 38. Payment Domain

Payment is initially intentionally simple.

Conceptual model:

```text
Payment
├── Tenant
├── Appointment
├── Amount
├── Method
├── Status
├── PaidAt
└── Reference
```

Methods:

```text
Cash
Card
Online
Other
```

Shinera MVP is not a full accounting system.

---

# 39. Dashboard Architecture

Dashboard is a read-model/reporting surface.

It should not duplicate core business state.

Dashboard queries aggregate data from:

```text
Appointments
Customers
Payments
Services
Staff
Branches
```

Typical outputs:

```text
Today's Appointments
Upcoming Appointments
Revenue
New Customers
Top Services
```

Dashboard APIs should favor purpose-built projections over loading domain graphs.

---

# 40. Public Booking Architecture

Public booking should reuse core domain services/rules.

Do not create a second appointment engine.

Conceptual flow:

```text
Public Business Page
↓
Branch
↓
Service
↓
Staff / Any Staff
↓
Date
↓
Available Slots
↓
Identify Customer
↓
Create Appointment
```

Additional public endpoint requirements:

```text
Rate limiting
Abuse protection
Safe validation
Limited data exposure
```

---

# 41. Notifications Architecture

Notification infrastructure should be abstracted.

Conceptual interface:

```text
INotificationSender
```

Possible providers:

```text
SMS
Email
Push
Messenger Integration
```

Core business logic should not depend directly on one notification vendor.

Do not overbuild notification infrastructure before appointment flows are stable.

---

# 42. Audit Architecture

Important actions should create audit records.

Conceptual:

```text
AuditLog
├── Tenant
├── Branch
├── User
├── Action
├── Entity
├── EntityId
└── Timestamp
```

High-value audited operations:

```text
Appointment creation
Appointment reschedule
Appointment cancellation
Appointment completion
Payment changes
Staff schedule changes
Permission changes
```

Audit is different from application logging.

Audit answers:

```text
Who changed business data?
```

Logging answers:

```text
What happened technically?
```

---

# 43. Soft Delete

Soft delete may apply to entities where historical references matter.

Candidates:

```text
Customer
Service
Staff
Branch
```

Soft delete should be implemented consistently through:

```text
DeletedAt / IsDeleted
+
Query Filters
+
Business rules
```

Exact strategy must be verified against the current repository.

---

# 44. Cross-Cutting Interceptors

EF Core interceptors may be used for concerns such as:

```text
Audit fields
Soft delete
Tenant stamping
```

Cross-cutting automation must not hide important business behavior.

For example:

```text
Automatically assigning TenantId
```

may be useful, but authorization must still validate ownership.

---

# 45. Validation Architecture

Validation has multiple levels.

```text
Transport validation
↓
Application validation
↓
Domain invariant
↓
Database constraint
```

Examples:

Transport:

```text
Required string
Format
Range
```

Application:

```text
Referenced entity exists
Entity belongs to tenant
Staff can provide service
```

Domain:

```text
Invalid appointment transition
```

Database:

```text
Uniqueness
Foreign key integrity
```

Do not force every rule into one validation layer.

---

# 46. Error Architecture

Errors should use stable machine-readable codes.

Examples:

```text
appointment.conflict
appointment.invalid_transition
staff.not_available
subscription.feature_unavailable
tenant.access_denied
```

Conceptual response:

```json
{
  "success": false,
  "error": {
    "code": "appointment.conflict",
    "message": "..."
  }
}
```

Frontend should react to:

```text
error.code
```

not parse natural-language text.

---

# 47. Global Exception Handling

Unexpected exceptions should be handled centrally.

Responsibilities:

```text
Log exception
Assign trace context
Return safe response
Hide internal details in production
```

Business errors should preferably not use global exception handling as ordinary control flow.

---

# 48. Frontend Architecture

Frontend technology:

```text
Angular 20
Angular Material
Trezo
```

Key characteristics:

```text
Standalone-oriented modern Angular
RTL
Persian-first UI
Dark / Light theme
Responsive web app
```

---

# 49. Frontend Module / Feature Boundaries

Suggested high-level areas:

```text
auth
workspace
dashboard
appointments
customers
staff
services
payments
settings
subscription
public-booking
shared
core
```

Exact Angular folder layout should follow the current repository conventions.

---

# 50. Frontend Core Layer

`core` may contain singleton/application-wide concerns such as:

```text
Authentication
HTTP interceptors
Current workspace
Permission service
Feature entitlement service
Global error handling
Configuration
```

Avoid placing generic reusable visual components in `core`.

---

# 51. Frontend Shared Layer

`shared` may contain:

```text
Reusable components
Pipes
Directives
Common UI helpers
Shared models
```

Do not turn `shared` into an uncontrolled dumping ground.

---

# 52. API Client Architecture

Frontend should use a consistent API client strategy.

Existing Scalar/OpenAPI client generation work may be used where appropriate.

Goals:

```text
Typed contracts
Enum consistency
Less duplicated DTO definition
Predictable error handling
```

Generated code should not be manually edited if it is intended to be regenerated.

---

# 53. Frontend Authorization

Frontend authorization exists for UX.

Conceptual:

```text
hasPermission(...)
```

or equivalent directive/service.

Use for:

```text
Routes
Buttons
Menus
Actions
```

Backend remains authoritative.

---

# 54. Frontend Feature Gating

Conceptual:

```text
hasFeature(...)
```

Use to:

```text
Hide unavailable feature
Disable action
Show upgrade CTA
```

Do not hardcode plan names across components.

---

# 55. Theme Architecture

Current Shinera visual identity uses a dusty-rose palette.

Primary design tokens include:

```text
--primaryColor: #a65d6d
--primaryHoverColor: #934f60
--headingColor: #27232a
--bodyColor: #737078
--borderColor: #e8e0e2
--successColor: #2e8b57
--errorColor: #d64545
--warningColor: #d99a2b
```

Primary font:

```text
Shabnam
```

Theme behavior must support:

```text
Light
Dark
RTL
```

Hardcoded component colors should be minimized where theme variables exist.

---

# 56. Localization Architecture

Display locale:

```text
fa-IR
```

Direction:

```text
RTL
```

Date strategy:

```text
Persistence:
Gregorian / normalized date-time

Display:
Persian calendar
```

Do not persist Jalali-formatted strings as canonical dates.

---

# 57. Date / Time Architecture

Date/time handling must explicitly distinguish:

```text
Date-only business concepts
Time-only schedule concepts
UTC timestamps
Local display
```

Examples:

```text
Appointment business date
Staff working hour
CreatedAt
PaidAt
```

These are not necessarily the same type of temporal concept.

Exact timezone strategy should be recorded in an ADR before Appointment implementation becomes deep.

Recommended ADR:

```text
ADR-005 Date and Time Strategy
```

---

# 58. Security Architecture

Main security boundaries:

```text
Authentication
Tenant isolation
Branch isolation
Authorization
Subscription entitlement
Public endpoint exposure
Token lifecycle
```

Never assume an ID is safe because the frontend supplied it.

Every protected resource should be evaluated against:

```text
Who is the user?
Which tenant?
Which branch?
Which permission?
Which scope?
Which feature?
```

---

# 59. Rate Limiting

High-risk/public endpoints should support rate limiting.

Candidates:

```text
Login
Registration
Password reset
Public booking
Public availability
Captcha
```

Rate limits should be configured based on endpoint sensitivity.

---

# 60. Observability Architecture

Baseline:

```text
Structured Logging
Health Checks
Tracing
Exception monitoring
Slow request visibility
```

Useful correlation fields:

```text
TraceId
TenantId
UserId
RequestPath
```

Sensitive data must be excluded.

The separate reusable AppMonitoring initiative may later be integrated, but it must not block Shinera core MVP.

---

# 61. Health Architecture

Expected endpoints:

```text
/health/live
/health/ready
```

Liveness:

```text
Is process alive?
```

Readiness:

```text
Can application serve real traffic?
```

Database or infrastructure dependencies may participate in readiness checks.

---

# 62. Testing Architecture

Test layers:

```text
Domain Tests
Application Tests
Integration Tests
E2E Tests
```

## Domain Tests

For:

```text
Business invariants
State transitions
Pure calculations
```

## Application Tests

For:

```text
Commands
Queries
Validation
Result behavior
```

## Integration Tests

For:

```text
EF Core
Database behavior
Tenant isolation
Authorization
Transactions
Endpoints
```

## E2E

Tool:

```text
Playwright
```

Use for critical user flows.

---

# 63. Critical E2E Flow

Primary MVP journey:

```text
Register / Login
↓
Create Service
↓
Create Staff
↓
Assign Service
↓
Configure Schedule
↓
Create Customer
↓
Calculate Available Slot
↓
Create Appointment
↓
Complete Appointment
↓
Register Payment
↓
Dashboard reflects data
```

This is the architectural integration test of the product as a whole.

---

# 64. Transaction Boundaries

Use transactions for operations that must remain atomic.

Critical example:

```text
Registration / workspace provisioning
```

Possible flow:

```text
User
Tenant
BusinessProfile
Branch
Membership
Subscription
Owner role
```

If one critical step fails, partial provisioning should not remain.

Appointment creation may also require transactional/concurrency coordination.

---

# 65. Concurrency

Concurrency is especially important for Appointment booking.

Risk:

```text
Request A checks 10:00 → free
Request B checks 10:00 → free
Request A saves
Request B saves
```

This must not result in an invalid double booking.

Exact strategy is not finalized in this document.

Possible approaches may include:

```text
database constraint strategy
transaction isolation
optimistic concurrency
locking / serialized booking section
```

Decision belongs in:

```text
ADR-004 Appointment Concurrency Strategy
```

---

# 66. Caching

Caching is not a default solution for all queries.

Potential future candidates:

```text
Plan/Feature catalog
Permission catalog
Static configuration
Expensive dashboard aggregates
```

Caching should only be introduced when:

```text
benefit is measurable
invalidation is understood
tenant boundaries are safe
```

Do not cache tenant-sensitive data without careful key design.

---

# 67. Background Processing

Background jobs may eventually support:

```text
Notifications
Reminders
Subscription expiration
Periodic cleanup
Analytics aggregation
```

Do not introduce a heavy job platform until a real requirement exists.

Background processing architecture should be selected when the first production use case is ready.

---

# 68. File Storage

Potential file use cases:

```text
Business logo
Staff image
Customer image
Attachments
```

Storage should be abstracted from business logic.

Possible implementation:

```text
IFileStorage
```

Local development and production storage may differ.

File validation is required for public/user uploads.

---

# 69. Deployment Architecture

Backend and frontend have independent repositories and deployment lifecycles.

Conceptually:

```text
Frontend
   ↓ HTTPS
Backend API
   ↓
Database
```

This separation supports independent:

```text
Build
Release
Rollback
Scaling
```

It does not require microservices.

---

# 70. CI/CD Architecture

Target backend pipeline:

```text
Restore
↓
Build
↓
Unit/Application Tests
↓
Integration Tests
↓
Publish
```

Target frontend pipeline:

```text
Install
↓
Lint / Validation
↓
Build
↓
Unit Tests
↓
Playwright
↓
Publish
```

Cross-stack release validation may run separately.

---

# 71. Environment Configuration

Configuration must be environment-driven.

Examples:

```text
Database connection
OpenIddict settings
Token configuration
CORS
Frontend API base URL
Logging
Storage
Notification providers
```

Secrets must not be committed.

Environment-specific behavior should not be hardcoded into source.

---

# 72. Architectural Non-Goals

Current architecture intentionally avoids:

```text
Microservices
Event sourcing
CQRS with separate physical databases
Distributed transactions
Kafka-based architecture
Complex service mesh
Premature DDD ceremony
Generic enterprise framework creation
```

These may only be introduced later with a demonstrated need.

---

# 73. Architecture Quality Attributes

Primary architecture goals:

## Security

Tenant isolation must be strong.

## Maintainability

Feature changes should remain localized.

## Testability

Business rules should be testable without UI.

## Performance

Ordinary SaaS operations should remain efficient.

## Reliability

Critical writes must preserve consistency.

## Evolvability

Solo → Salon → Multi-branch growth should not require redesigning the product core.

---

# 74. Known Architecture Decisions

Already established or strongly selected:

```text
ADR Candidate / Decision

Modular Monolith
Vertical Slice
CQRS
Minimal API
.NET 10
Angular 20
EF Core
OpenIddict
IDispatcher
Result Pattern
Scalar
Multi-Tenant
Tenant + optional Branch
Separate API/UI resource catalogs
Feature-based subscription
Gregorian persistence / Persian display
Separate frontend/backend repositories
Separate product repository
```

Some of these should be formalized as ADRs.

---

# 75. Recommended Initial ADRs

Create first:

```text
ADR-001 — Multi-Tenancy Strategy
ADR-002 — Authentication with OpenIddict
ADR-003 — Authorization and Permission Model
ADR-004 — Appointment Concurrency Strategy
ADR-005 — Date and Time Strategy
ADR-006 — Subscription and Feature Gating
ADR-007 — Staff vs User Identity Model
ADR-008 — Repository / Deployment Separation
```

Not all must be written immediately.

Prioritize ADRs that affect upcoming implementation.

---

# 76. Architecture Decision Rule

Create an ADR when the decision:

```text
affects multiple features
is costly to reverse
changes a major architectural pattern
changes persistence strategy
changes auth/security behavior
introduces major infrastructure
```

Do not create ADRs for trivial code-style decisions.

---

# 77. Current Architecture Risks

## 1. Tenant Isolation

Risk:

```text
Cross-tenant data exposure
```

Mitigation:

```text
CurrentTenant
global scoping
ownership validation
integration tests
```

## 2. Appointment Concurrency

Risk:

```text
Double booking
```

Mitigation:

```text
Backend final validation
database-safe concurrency strategy
tests
```

## 3. Permission Complexity

Risk:

```text
Role + Scope + Branch + Ownership rules become scattered
```

Mitigation:

```text
central permission model
dynamic authorization
stable permission catalog
```

## 4. Subscription Duplication

Risk:

```text
Plan-name checks spread across code
```

Mitigation:

```text
Feature capability checks
```

## 5. Date / Time

Risk:

```text
Persian display logic leaks into persistence/domain
```

Mitigation:

```text
clear temporal strategy
ADR
central conversion utilities
```

## 6. Dashboard Before Core Domain

Risk:

```text
UI progresses faster than real business data
```

Mitigation:

```text
prioritize core vertical slices before further dashboard polish
```

---

# 78. Target Architecture Flow

A typical authenticated request:

```text
Angular
  ↓
HTTPS
  ↓
Minimal API
  ↓
Authentication
  ↓
Current User / Tenant / Branch
  ↓
Authorization
  ↓
Feature Entitlement
  ↓
Command / Query
  ↓
Handler
  ↓
Domain Rules
  ↓
IApplicationDbContext
  ↓
EF Core
  ↓
Database
```

Response:

```text
Database
  ↓
Handler
  ↓
Result<T>
  ↓
ApiResponse<T>
  ↓
HTTP
  ↓
Angular
```

---

# 79. Appointment Creation Architecture Flow

```text
Angular Appointment Form
        ↓
Create Appointment Request
        ↓
Authentication
        ↓
Tenant / Branch Context
        ↓
Permission Check
        ↓
Feature Check
        ↓
CreateAppointmentCommand
        ↓
Validate Customer
        ↓
Validate Service
        ↓
Validate Staff
        ↓
Validate Staff-Service
        ↓
Validate Staff-Branch
        ↓
Resolve Effective Schedule
        ↓
Check Break / Time Off
        ↓
Check Existing Appointment
        ↓
Concurrency Protection
        ↓
Create Appointment
        ↓
Commit Transaction
        ↓
Result
        ↓
UI Refresh
```

This flow represents the core architectural value chain of Shinera.

---

# 80. Registration Architecture Flow

```text
Landing
  ↓
Plan Selection
  ↓
Registration Wizard
  ↓
Registration Command
  ↓
Validate Input
  ↓
Begin Transaction
  ↓
Create User
  ↓
Create Tenant
  ↓
Create Business Profile
  ↓
Create Main Branch
  ↓
Create Membership
  ↓
Create Subscription
  ↓
Assign Owner Permission/Role
  ↓
Commit
  ↓
Authenticate
  ↓
Workspace
  ↓
Onboarding
```

---

# 81. Dashboard Architecture Flow

```text
Dashboard
  ↓
Dashboard Query
  ↓
Current Tenant / Branch
  ↓
Date Filters
  ↓
Purpose-built projections
  ↓
Aggregations
  ↓
Dashboard DTO
  ↓
Widgets
```

Avoid loading large domain graphs for dashboard aggregation.

---

# 82. Architecture Governance

Architecture is governed by:

```text
SHINERA-PRD.md
SHINERA-PRODUCT-BACKLOG.md
PROJECT-INSTRUCTIONS.md
DEFINITION-OF-DONE.md
ARCHITECTURE-OVERVIEW.md
ADRs
Current repositories
```

When architecture documentation and repository implementation diverge:

```text
1. Identify the divergence.
2. Determine whether code or documentation is outdated.
3. Update the correct side.
4. Record major decisions in ADR.
```

Do not let architecture documents become aspirational fiction.

---

# 83. Current Architecture Summary

Shinera is designed as:

```text
A modular multi-tenant SaaS
with a .NET 10 backend
and Angular 20 frontend,
organized around vertical business slices,
using CQRS, Minimal API, EF Core,
OpenIddict authentication,
dynamic permission authorization,
feature-based subscriptions,
and strict Tenant/Branch isolation.
```

The architecture deliberately stays simpler than a distributed enterprise platform while preserving strong boundaries around:

```text
Tenancy
Security
Appointments
Subscription
Business rules
```

The next architecture work should focus on the decisions that directly block upcoming MVP implementation:

```text
Multi-Tenancy
Authorization
Appointment Concurrency
Date / Time
Subscription Feature Gating
```
