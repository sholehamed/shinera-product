# Shinera — Project Instructions

**Project:** Shinera  
**Document Role:** Project-wide execution and collaboration rules  
**Audience:** ChatGPT, Work, coding agents, reviewers, and contributors  
**Status:** Active  
**Version:** 1.0

---

# 1. Purpose

This document defines how work on Shinera must be performed.

It is not a Product Requirements Document and it does not replace the Product Backlog.

Use this document to answer:

- How should work be approached?
- Which source has authority when documents disagree?
- How should backend and frontend work be coordinated?
- What must be checked before implementing a feature?
- What quality gates must be satisfied?
- What changes require an explicit architectural or product decision?

---

# 2. Project Repositories

Shinera is intentionally split into three repositories.

```text
shinera-product
shinera-backend
shinera-frontend
```

## 2.1 shinera-product

Purpose:

- Product requirements
- Product backlog
- Product decisions
- Architecture decisions
- Development status
- Definition of Done
- Cross-repository documentation

This repository is the source of truth for **what should be built and why**.

Recommended structure:

```text
shinera-product/
├── README.md
└── docs/
    ├── product/
    │   ├── SHINERA-PRD.md
    │   └── SHINERA-PRODUCT-BACKLOG.md
    │
    ├── development/
    │   ├── PROJECT-INSTRUCTIONS.md
    │   ├── DEVELOPMENT-STATUS.md
    │   └── DEFINITION-OF-DONE.md
    │
    └── architecture/
        ├── ARCHITECTURE-OVERVIEW.md
        └── decisions/
```

## 2.2 shinera-backend

Purpose:

- .NET backend implementation
- Domain model
- Application layer
- Infrastructure
- API
- Database migrations
- Backend tests

This repository is the source of truth for **the current backend implementation**.

## 2.3 shinera-frontend

Purpose:

- Angular application
- UX implementation
- Routes
- Components
- Forms
- Client-side permissions
- Public booking UI
- Frontend tests and Playwright E2E tests

This repository is the source of truth for **the current frontend implementation**.

---

# 3. Source of Truth Hierarchy

When information conflicts, use the following authority order.

```text
1. Current repository implementation
2. Explicit current task / requirement
3. Approved product decisions
4. SHINERA-PRODUCT-BACKLOG.md
5. SHINERA-PRD.md
6. PROJECT-INSTRUCTIONS.md
7. Backend / Frontend project guides
8. Existing architectural decisions (ADR)
9. Generic best practices
```

Important:

The current repository is authoritative for **what currently exists**, not automatically for **what is product-correct**.

If the repository contradicts an explicit approved requirement:

- Do not silently preserve the wrong behavior.
- Do not silently rewrite unrelated architecture.
- Identify the conflict.
- Implement the smallest correct change.
- Record a decision if the change has architectural impact.

---

# 4. Core Working Principle

Never start implementation from assumptions.

Before modifying code:

1. Inspect the relevant repository.
2. Find similar existing features.
3. Identify current conventions.
4. Read the relevant product requirement or backlog story.
5. Check dependencies and business rules.
6. Check authorization and tenant isolation implications.
7. Check whether the change affects backend, frontend, database, or tests.
8. Only then implement.

Do not introduce a new pattern when an established project pattern already exists unless there is a strong reason.

---

# 5. Product Scope Rules

Shinera is a multi-tenant SaaS for beauty businesses and solo professionals.

The MVP focuses on:

```text
Authentication
Tenant / Workspace
Branch
Subscription / Feature Gating
Authorization
Business Profile
Services
Staff
Schedules
Customers
Appointments
Payments
Dashboard
```

Features outside the currently approved MVP must not be introduced opportunistically.

Examples:

```text
Force Appointment
VIP business rules
Loyalty
Advanced reporting
Marketing automation
Marketplace
Payroll
Inventory
```

These may exist in the roadmap, but roadmap presence does not mean implementation permission.

---

# 6. Product Decision Rules

A product decision is required when a task changes:

- MVP scope
- Pricing or plan behavior
- User flow
- Appointment business rules
- Subscription rules
- VIP behavior
- Cancellation policy
- Public booking behavior
- Roles or permission semantics
- Tenant or branch behavior

Do not make hidden product decisions inside implementation code.

If a requirement is ambiguous:

- Prefer the smallest implementation consistent with the PRD and backlog.
- Avoid expanding scope.
- Record unresolved product questions rather than inventing complex behavior.

---

# 7. Architecture Principles

Shinera backend follows:

```text
.NET 10
Modular Monolith
Vertical Slice Architecture
CQRS
Minimal API
EF Core
OpenIddict
Result Pattern
Multi-Tenancy
```

Frontend follows:

```text
Angular 20
Angular Material
Trezo-based UI
RTL
Persian-first UX
Dark / Light Theme
```

Architecture should optimize for:

- Maintainability
- Clear business boundaries
- Testability
- Tenant safety
- Incremental growth
- Low accidental complexity

Avoid premature microservices.

---

# 8. Backend Rules

## 8.1 Vertical Slice First

A feature should preferably keep its application logic together.

Typical slice:

```text
Feature
├── Command / Query
├── Handler
├── Validator
├── DTO
├── Mapping
└── Endpoint
```

Follow the repository's existing structure if it differs.

## 8.2 CQRS

Commands mutate state.

Queries read state.

Do not mix unrelated read and write responsibilities in one handler.

## 8.3 Dispatcher

Use the current Shinera dispatcher abstraction.

Do not introduce MediatR or another dispatcher if the repository already uses `IDispatcher`.

## 8.4 Result Pattern

Expected business failures should use the project Result pattern.

Examples:

- validation failure
- conflict
- business rule violation
- missing allowed resource

Exceptions are reserved for truly exceptional or infrastructure-level conditions unless the project already defines otherwise.

## 8.5 Minimal API

Use the repository's existing Minimal API conventions.

Endpoints should remain thin.

Business logic belongs in the appropriate application/domain layer.

## 8.6 API Responses

Follow the current `ApiResponse` / `ApiResponse<T>` convention.

Do not invent a new API envelope without an explicit decision.

---

# 9. Domain Rules

Business invariants must not live only in the UI.

Important business behavior should be enforced in the backend and, when appropriate, inside domain behavior.

Examples:

- Appointment state transitions
- Staff availability
- Appointment conflicts
- Tenant ownership
- Branch access
- Feature availability
- Subscription restrictions

Prefer explicit business rules over scattered boolean conditions.

---

# 10. Multi-Tenancy Rules

Tenant isolation is a security boundary.

## Required Rules

- Tenant-specific entities must be associated with a Tenant.
- Client-provided `TenantId` must never be trusted as proof of access.
- Current tenant context should determine tenant-scoped operations.
- Cross-tenant access must be impossible by changing IDs in a request.
- Tenant filters must apply consistently.
- Commands and queries must validate related entities belong to the same tenant.

## Critical Testing

For important tenant-scoped features, include tests proving:

```text
Tenant A cannot read Tenant B data
Tenant A cannot update Tenant B data
Tenant A cannot delete Tenant B data
Tenant A cannot reference Tenant B entities in commands
```

Any tenant isolation failure is considered a critical defect.

---

# 11. Branch Rules

`TenantId` is required for tenant-owned entities.

`BranchId` is optional only where the business model allows it.

A user having Tenant access does not automatically imply access to every Branch.

Branch-sensitive operations must respect:

- Current branch
- Branch membership
- Permission scope
- Feature availability

Main Branch must always remain valid according to current business rules.

---

# 12. Authorization Rules

Authorization must be enforced by the backend.

Frontend visibility is UX, not security.

The authorization model is based on:

```text
Resource
Action
Scope
```

Scopes may include:

```text
Tenant
Branch
Own
Child
```

Only use scopes currently supported by the implementation.

When adding a protected feature:

1. Define or reuse the correct permission.
2. Enforce it in backend.
3. Apply corresponding UI visibility/availability.
4. Test unauthorized access.

---

# 13. Subscription and Feature Gating

Feature access must not depend only on hidden frontend elements.

For gated functionality:

```text
Frontend
→ hide / disable unavailable feature

Backend
→ enforce entitlement
```

Plan / subscription logic must be centralized enough to avoid duplicated plan checks across the codebase.

Prefer checking a feature capability over hardcoding plan names.

Good:

```text
Feature.MultipleBranches
```

Avoid:

```text
if (plan == "SalonPro")
```

unless the current architecture explicitly requires it.

---

# 14. Appointment Domain Rules

Appointments are the core business domain of Shinera.

Changes to appointment logic require extra care.

Before creating or rescheduling an appointment, validate at minimum:

- Tenant
- Branch
- Customer
- Service
- Staff
- Staff active state
- Service active state
- Staff-to-service assignment
- Staff-to-branch assignment
- Work schedule
- Breaks
- Time off where supported
- Existing appointment overlap
- Appointment state rules
- Feature / permission requirements

Do not rely on frontend slot availability as proof that a slot is still valid.

The backend must perform the final availability check.

Concurrency must be considered whenever two requests could reserve the same slot.

---

# 15. Date and Time Rules

Storage and display must remain separate concerns.

Recommended project rule:

```text
Storage:
Gregorian / UTC where appropriate

Display:
fa-IR / Persian calendar
```

Do not store formatted Persian dates as domain date values.

Do not mix display conversion logic into persistence logic.

Timezone assumptions must be explicit when they affect business behavior.

---

# 16. Database Rules

When changing the database:

1. Update entity/domain model.
2. Update EF configuration.
3. Add required indexes.
4. Add constraints where useful.
5. Generate migration.
6. Review generated migration.
7. Test upgrade path.

Important constraints should not exist only in application code when the database can safely enforce them.

However, do not use database constraints as a replacement for clear business validation.

---

# 17. Query Rules

List endpoints should follow consistent patterns.

Where appropriate support:

```text
Page
PageSize
Search
Sort
Filters
```

Avoid:

- loading entire tables into memory
- unnecessary `Include`
- N+1 queries
- fetching full entities when projection is sufficient

Use projection for read models when practical.

Do not compile every EF query by default.

Optimize based on evidence and hot paths.

---

# 18. Frontend Rules

## 18.1 Existing Design System

Follow the existing Shinera visual system and Trezo/Angular Material structure.

Do not introduce unrelated UI libraries without approval.

## 18.2 UX States

Every meaningful data-driven page should consider:

```text
Loading
Empty
Success
Validation Error
API Error
Unauthorized
Disabled Feature
```

## 18.3 RTL

Persian UI must be checked in RTL.

Do not assume a component is RTL-safe because text alignment looks correct.

Check:

- icons
- spacing
- arrows
- menus
- forms
- tables
- pagination
- date controls

## 18.4 Dark Mode

New UI should work with the project's existing dark/light theme behavior.

Avoid hardcoded colors when theme variables already exist.

## 18.5 Responsive Behavior

Desktop is important, but new pages must not become unusable on tablet/mobile widths.

## 18.6 API Integration

Use the existing API client/service generation conventions.

Do not create one-off HTTP patterns if a shared client infrastructure exists.

---

# 19. Backend + Frontend Feature Contract

A feature is not complete merely because one repository is implemented.

For cross-stack features, define the contract before or during implementation:

```text
Request
Response
Validation Errors
Business Errors
Permissions
Feature Gates
Loading Behavior
Empty State
Success Behavior
```

Backend and frontend must agree on enum/value semantics.

Avoid duplicating business logic independently on both sides.

---

# 20. Testing Rules

Testing is part of implementation, not a follow-up task.

## 20.1 Domain Tests

Use for:

- Business invariants
- State transitions
- Pure domain rules

## 20.2 Application Tests

Use for:

- Command handlers
- Query handlers
- Validation
- Result behavior

## 20.3 Integration Tests

Required for critical infrastructure and business flows.

High-priority examples:

```text
Authentication
Tenant isolation
Authorization
Service creation
Staff assignment
Customer creation
Appointment availability
Appointment conflict
Appointment reschedule
Appointment cancellation
Appointment completion
Payment registration
```

## 20.4 E2E Tests

Use Playwright for critical user journeys.

Primary golden path:

```text
Login
→ Create Service
→ Create Staff
→ Configure Schedule
→ Create Customer
→ Create Appointment
→ Complete Appointment
→ Register Payment
```

Registration flow should have its own E2E coverage.

---

# 21. Security Rules

Security-sensitive changes must consider:

- Authentication
- Authorization
- Tenant isolation
- Branch isolation
- Token lifecycle
- Input validation
- Rate limiting
- Sensitive logging
- File upload validation
- Public endpoint abuse

Never log:

- Passwords
- Access tokens
- Refresh tokens
- Secrets
- Sensitive authentication payloads

Do not expose internal exception details to clients in production.

---

# 22. Observability Rules

Important requests should be traceable.

Where supported, logs should include useful context such as:

```text
TraceId
TenantId
UserId
RequestPath
```

Do not log sensitive data merely for debugging convenience.

Important failures should be diagnosable without reproducing them locally.

---

# 23. Implementation Workflow

For every backlog story, follow this sequence.

```text
1. Identify Story ID
2. Read story and acceptance criteria
3. Inspect current implementation
4. Identify dependencies
5. Identify affected repositories
6. Identify domain/business rules
7. Identify authorization requirements
8. Identify subscription/feature gates
9. Design data/API changes
10. Implement backend
11. Implement frontend if required
12. Add/update tests
13. Build affected projects
14. Run relevant test suites
15. Review for tenant/security issues
16. Review UX states
17. Update development status
18. Report completed work and remaining risks
```

Do not skip directly from requirement to code generation.

---

# 24. Feature Completion Checklist

For each feature, evaluate:

```text
Domain
├── Entity / behavior
├── Business rules
└── State transitions

Database
├── Configuration
├── Constraints
├── Indexes
└── Migration

Backend
├── Command / Query
├── Handler
├── Validator
├── DTO
├── Endpoint
└── Error mapping

Authorization
├── Permission
├── Scope
└── Feature gate

Frontend
├── Route
├── Page / Component
├── API integration
├── Form validation
├── Loading state
├── Empty state
├── Error state
├── RTL
├── Dark mode
└── Responsive behavior

Testing
├── Domain
├── Application
├── Integration
└── E2E when critical
```

Not every feature requires every item, but every item must be consciously evaluated.

---

# 25. Definition of Done

A story may only be considered Done when it satisfies the project's `DEFINITION-OF-DONE.md`.

Until that document exists, use the following minimum:

```text
✓ Requirement implemented
✓ Business rules enforced
✓ Backend authorization enforced
✓ Tenant isolation verified
✓ Validation implemented
✓ Database migration added if needed
✓ Frontend states handled if UI changed
✓ RTL checked if UI changed
✓ Dark mode checked if UI changed
✓ Relevant tests added
✓ Build succeeds
✓ Critical tests pass
✓ No known P0/P1 defect remains
```

---

# 26. Development Status

`DEVELOPMENT-STATUS.md` must represent what is actually implemented.

Do not mark work completed based only on:

- a generated file
- an untested code path
- an unfinished frontend
- a stub
- a TODO
- an assumed migration

Status should distinguish where useful:

```text
Not Started
In Progress
Backend Done
Frontend Done
Integration Pending
Testing
Done
Blocked
```

---

# 27. Architecture Decisions

Create an ADR when a decision:

- changes a major architectural pattern
- affects multiple features
- creates a long-term constraint
- changes persistence strategy
- changes authentication/authorization model
- introduces significant infrastructure
- would be expensive to reverse later

Examples:

```text
ADR-001 Multi-Tenancy Strategy
ADR-002 OpenIddict Authentication
ADR-003 Permission Model
ADR-004 Appointment Concurrency Strategy
ADR-005 Date and Time Strategy
```

Do not create ADRs for trivial implementation choices.

---

# 28. Refactoring Rules

Do not perform large unrelated refactors while implementing a backlog story.

Allowed:

- Small local cleanup required by the change
- Removing duplication directly introduced/exposed by the feature
- Correcting an obvious nearby defect

Requires separate task or explicit approval:

- Renaming large modules
- Replacing architectural patterns
- Reorganizing entire folders
- Swapping major libraries
- Broad database redesign
- Rewriting unrelated features

Keep diffs focused.

---

# 29. Dependency Rules

Before adding a package:

1. Verify existing project capability.
2. Check whether a current dependency already solves the problem.
3. Prefer stable, maintained packages.
4. Avoid adding packages for trivial utilities.
5. Consider backend/frontend bundle or runtime impact.
6. Record significant infrastructure dependencies.

Do not introduce competing libraries for the same concern without justification.

---

# 30. Code Quality Rules

Prefer:

- Explicit names
- Small focused methods
- Clear business terminology
- Predictable error codes
- Consistent conventions
- Readable code over clever code

Avoid:

- magic strings
- duplicated business rules
- hidden side effects
- massive handlers
- generic abstractions with no real reuse
- premature optimization
- speculative architecture

---

# 31. Error Handling

Business errors should use stable machine-readable error codes.

Example:

```text
appointment.conflict
appointment.invalid_transition
staff.not_available
subscription.feature_unavailable
tenant.access_denied
```

Frontend should not need to parse natural-language messages to determine behavior.

Human-readable messages may change.

Error codes should remain stable unless intentionally versioned.

---

# 32. Review Rules

When reviewing a Shinera change, evaluate at least:

1. Product correctness
2. Business rules
3. Tenant isolation
4. Authorization
5. Feature gating
6. Data integrity
7. Concurrency
8. Performance
9. Error handling
10. Test coverage
11. Maintainability
12. UX states
13. RTL / theme behavior where relevant

Review findings should be classified when useful:

```text
Critical
High
Medium
Low
```

---

# 33. Work / Agent Behavior

When an AI agent is asked to implement work:

## It should:

- Inspect before modifying
- Reuse established patterns
- Keep changes scoped
- Build after changes
- Run relevant tests
- Report what changed
- Report tests run
- Report unresolved risks
- Update documentation/status when required

## It should not:

- Invent missing project structure
- Pretend tests passed if not run
- Rewrite architecture silently
- Add unrelated features
- Trust frontend validation for security
- Mark incomplete work Done
- Hide important tradeoffs
- Replace existing conventions merely because another pattern is popular

---

# 34. Chat Role Separation

Recommended ChatGPT Project chats:

```text
00 — Product & Planning
01 — Architecture
02 — Backend
03 — Frontend
04 — Review & QA
```

## 00 — Product & Planning

Use for:

- PRD
- Backlog
- MVP scope
- Business rules
- Prioritization
- User flows
- Roadmap

## 01 — Architecture

Use for:

- Multi-tenancy
- Authorization
- OpenIddict
- Concurrency
- Domain boundaries
- Persistence
- Cross-cutting concerns
- ADRs

## 02 — Backend

Use for:

- Commands / Queries
- EF Core
- APIs
- Domain implementation
- Backend tests
- Debugging

## 03 — Frontend

Use for:

- Angular
- UI/UX implementation
- API integration
- Forms
- RTL
- Theme
- Playwright

## 04 — Review & QA

Use for:

- PR review
- Security review
- Test planning
- Regression review
- Release readiness

If a discussion clearly belongs to another chat, move the work there rather than mixing unrelated contexts.

---

# 35. Recommended Task Start Format

A task should ideally start with:

```text
Story: SHN-XXX
Goal: <short goal>
Repository: backend / frontend / both
References:
- SHINERA-PRD.md
- SHINERA-PRODUCT-BACKLOG.md

Instruction:
Inspect the current implementation first.
Follow existing Shinera architecture and conventions.
Implement only the approved scope.
Add/update relevant tests.
Build and report the result.
```

For mature workflow, saying:

```text
Start SHN-132
```

should be sufficient when the project context and repositories are available.

---

# 36. Golden Rule

The goal is not to generate the most code.

The goal is to move Shinera forward with:

```text
Correct product behavior
+
Safe tenant isolation
+
Consistent architecture
+
Good UX
+
Reliable tests
+
Small understandable changes
```

Every implementation decision should support that goal.
