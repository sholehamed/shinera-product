# ADR-001 — Multi-Tenancy Strategy

**Project:** Shinera  
**Status:** Accepted  
**Date:** 2026-10-05  
**Decision Type:** Architecture / Security / Data Isolation  
**Scope:** Backend, Database, Authorization, Workspace, Testing  
**Related Documents:**  
- `SHINERA-PRD.md`
- `SHINERA-PRODUCT-BACKLOG.md`
- `PROJECT-INSTRUCTIONS.md`
- `ARCHITECTURE-OVERVIEW.md`
- `DEFINITION-OF-DONE.md`

---

# 1. Context

Shinera is a SaaS platform serving multiple independent beauty businesses.

Each business must operate in an isolated workspace with its own:

```text
Business Profile
Branches
Users / Memberships
Staff
Services
Customers
Appointments
Payments
Subscription
Settings
```

The architecture must guarantee that data belonging to one business cannot be accessed by another business.

This is not only a data modeling concern.

Multi-tenancy affects:

```text
Authentication
Authorization
Database queries
Commands
Background jobs
Caching
Logging
Audit
File storage
Testing
Public booking
Dashboard aggregation
```

A multi-tenant mistake is considered a security defect.

---

# 2. Decision

Shinera will use:

```text
Shared Application
+
Shared Database
+
Shared Schema
+
TenantId discriminator
```

for tenant-owned business data.

Each business is represented by a:

```text
Tenant
```

Tenant-owned entities contain:

```text
TenantId
```

either directly or through a relationship that guarantees tenant ownership.

The application resolves the current Tenant through trusted authenticated context and enforces isolation in both:

```text
Data access
+
Business/application validation
```

Tenant identity supplied by a client request is never sufficient proof of authorization.

---

# 3. Selected Model

The selected topology is:

```text
                    Shinera Platform
                          │
              ┌───────────┴───────────┐
              │                       │
          Tenant A                Tenant B
              │                       │
       ┌──────┴──────┐         ┌──────┴──────┐
       │             │         │             │
   Branch A1      Branch A2  Branch B1     Branch B2
       │                       │
       └──── Tenant Data ──────┘
```

All tenants share the same application instance and database schema.

Isolation is performed by tenant ownership.

---

# 4. Why Shared Database / Shared Schema

The following alternatives were considered.

## Option A — Database per Tenant

```text
Tenant A → DB A
Tenant B → DB B
Tenant C → DB C
```

### Advantages

- Strong physical isolation
- Easier tenant-specific backup/restore
- Potential enterprise customization

### Disadvantages

- Higher operational complexity
- Harder migrations
- More connection management
- More expensive infrastructure
- Overkill for Shinera MVP and expected early scale

### Decision

Rejected for current architecture.

---

## Option B — Schema per Tenant

```text
Database
├── tenant_a.*
├── tenant_b.*
└── tenant_c.*
```

### Advantages

- Stronger logical separation than shared tables
- One database instance

### Disadvantages

- Migration complexity
- Schema lifecycle complexity
- Tooling complexity
- Difficult operational management as tenant count grows

### Decision

Rejected.

---

## Option C — Shared Schema with TenantId

```text
Appointments
├── Id
├── TenantId
├── ...
```

### Advantages

- Simple deployment
- Simple migrations
- Efficient SaaS operations
- Straightforward reporting
- Easy onboarding
- Lower infrastructure cost
- Fits Modular Monolith architecture

### Disadvantages

- Isolation must be implemented correctly
- Query mistakes can expose cross-tenant data
- Caching and background processing require discipline

### Decision

Accepted.

---

# 5. Tenant Entity

Conceptual Tenant:

```text
Tenant
├── Id
├── Name
├── Slug
├── Status
├── IsActive
├── CreatedAt
└── ...
```

Tenant represents the business workspace.

Tenant is not the same as:

```text
Branch
User
Subscription
BusinessProfile
```

These are related concepts.

---

# 6. Tenant-Owned Entities

Entities that belong to a specific business should normally contain `TenantId`.

Examples:

```text
Branch
BusinessProfile
Staff
ServiceCategory
Service
Customer
Appointment
Payment
Subscription
TenantMembership
BranchMembership
```

Other tenant-owned entities should follow the same rule.

---

# 7. Explicit TenantId Preference

For important aggregate/business entities, prefer explicit:

```text
TenantId
```

even when the Tenant could theoretically be inferred through another relationship.

Example:

```text
Appointment
├── TenantId
├── BranchId
├── CustomerId
├── StaffId
└── ServiceId
```

This provides:

- simpler filtering,
- safer queries,
- easier indexing,
- clearer ownership,
- easier diagnostics,
- easier audit/log correlation.

Do not force every low-level child entity to duplicate TenantId if ownership is already structurally guaranteed and duplication would add no value.

The decision should favor clarity and isolation.

---

# 8. Tenant Context

Shinera will expose a trusted current tenant abstraction:

```text
ICurrentTenant
```

Conceptually:

```csharp
public interface ICurrentTenant
{
    Guid TenantId { get; }
    Guid? BranchId { get; }
    Guid UserId { get; }
}
```

Exact implementation may differ.

The context is resolved from trusted authentication/workspace state.

It must not be populated directly from arbitrary request body fields.

---

# 9. Source of Tenant Identity

Tenant identity should come from authenticated/session/workspace context.

Trusted sources may include:

```text
Authenticated user membership
Validated workspace selection
Trusted token/session claims
Server-side context
```

Untrusted sources include:

```text
Request body TenantId
Query-string TenantId
Route TenantId
Custom browser header
Frontend local storage value
```

Untrusted values may be used as selectors only after server-side validation.

They are not authorization proof.

---

# 10. Client TenantId Rule

For normal tenant-scoped commands:

Avoid requiring:

```json
{
  "tenantId": "..."
}
```

when the Tenant can be resolved from current context.

Preferred:

```json
{
  "name": "Haircut"
}
```

Then:

```text
TenantId = currentTenant.TenantId
```

inside the application.

This reduces accidental or malicious cross-tenant references.

---

# 11. Administrator / Platform Exceptions

Platform-level administrative functionality may need to explicitly select a Tenant.

This is a separate privileged context.

Example:

```text
Platform Administrator
→ Select Tenant
→ Perform system operation
```

Such behavior must:

- use explicit platform permissions,
- not reuse ordinary tenant endpoints casually,
- be separately audited,
- not weaken normal Tenant isolation.

---

# 12. System Tenant

Shinera may maintain a System Tenant for platform-owned/bootstrap data.

Possible uses:

```text
Initial installation data
Internal platform records
System-level ownership where required
```

The System Tenant is not a shortcut for data that does not clearly belong somewhere.

Rules:

- Normal users cannot access it.
- Tenant filters must not accidentally expose it.
- Platform operations must use explicit system context.
- System Tenant data must be distinguishable from customer Tenant data.

---

# 13. Tenant Membership

Users may belong to one or more Tenants.

Conceptual model:

```text
User
   │
   ▼
TenantMembership
   │
   ▼
Tenant
```

A membership establishes:

```text
User belongs to Tenant
```

It does not automatically mean:

```text
User can do everything in Tenant
```

Permissions determine actions.

---

# 14. TenantMembership Responsibilities

Conceptually:

```text
TenantMembership
├── UserId
├── TenantId
├── Status
├── Role / RoleAssignment
└── ...
```

Membership is responsible for workspace relationship.

Authorization remains permission-driven.

---

# 15. Multi-Tenant User Support

Architecture should permit a User to belong to multiple Tenants.

Example:

```text
User
├── Salon A
└── Salon B
```

This allows future scenarios such as:

- consultant/manager working across businesses,
- owner of multiple businesses,
- shared specialist.

The MVP UI may simplify this flow, but the domain should not unnecessarily prevent it.

---

# 16. Workspace Selection

After authentication:

```text
User
 ↓
Available Tenant Memberships
 ↓
Tenant Selection
 ↓
Branch Selection
 ↓
Workspace
```

If only one Tenant exists:

```text
Tenant selection may be skipped
```

If only one accessible Branch exists:

```text
Branch selection may be skipped
```

The server remains responsible for validating both.

---

# 17. Branch Is Not Tenant

A Branch belongs to a Tenant.

```text
Tenant
 ├── Branch A
 └── Branch B
```

Branch is an operational scope, not a separate tenant.

Therefore:

```text
TenantId = security boundary
BranchId = operational/access scope
```

Branch data must never escape its parent Tenant.

---

# 18. BranchId Rule

`BranchId` is optional only where the business concept truly permits tenant-wide data.

Examples of Tenant-wide data:

```text
BusinessProfile
Subscription
Tenant settings
Possibly Service definition
```

Examples commonly Branch-related:

```text
Appointment
Staff assignment
Branch schedule
Payments associated with branch activity
```

Exact Branch ownership is defined per domain model.

---

# 19. Solo Mode

Solo businesses still use:

```text
Tenant
+
Main Branch
```

Do not create a separate persistence model for Solo.

Conceptually:

```text
Solo Tenant
  ↓
Main Branch
```

This keeps the model consistent if the business later grows into Salon mode.

---

# 20. Main Branch Rule

Every normal Tenant has a Main Branch.

During registration/workspace provisioning:

```text
Create Tenant
↓
Create Main Branch
```

The Main Branch should remain valid.

Operations that could leave the Tenant without a valid Main Branch must be rejected or handled explicitly.

---

# 21. Multi-Branch Feature

Multiple Branches may be controlled by subscription Feature Gating.

Conceptually:

```text
Feature.MultipleBranches
```

A Tenant without the feature still has:

```text
Main Branch
```

but cannot create/use additional branches beyond allowed limits.

Backend enforces this.

---

# 22. Data Query Strategy

Tenant-scoped queries must always include tenant isolation.

Conceptually:

```csharp
db.Customers
    .Where(x => x.TenantId == currentTenant.TenantId);
```

The exact implementation may use:

```text
Global Query Filters
Explicit filters
Query abstraction
Combination of both
```

The architectural goal is defense in depth.

---

# 23. Global Query Filters

EF Core Global Query Filters are appropriate for reducing accidental data exposure.

Conceptually:

```csharp
HasQueryFilter(x => x.TenantId == currentTenant.TenantId);
```

They may also interact with:

```text
Soft delete
```

Example conceptual filter:

```text
TenantId == CurrentTenant
AND
IsDeleted == false
```

---

# 24. Global Filter Limitation

Global Query Filters are not considered a complete security system.

Reasons include:

- filters can be disabled,
- raw SQL can bypass them,
- incorrect context can produce incorrect filtering,
- special system operations may intentionally ignore filters,
- related entity IDs still need validation.

Therefore:

```text
Global Query Filter
+
Application ownership validation
+
Authorization
+
Tests
```

is the intended strategy.

---

# 25. Query Filter Bypass

Any use of mechanisms such as:

```text
IgnoreQueryFilters()
Raw SQL
Direct database access
```

against tenant-owned data is security-sensitive.

It must be:

- intentional,
- reviewed,
- appropriately authorized,
- scoped,
- covered by tests when relevant.

Do not use filter bypass simply to make a query easier.

---

# 26. Write Isolation

Writes require stronger validation than simply stamping `TenantId`.

Example:

A request contains:

```text
ServiceId = X
StaffId = Y
```

The server must validate:

```text
Service X belongs to current Tenant
Staff Y belongs to current Tenant
```

before creating:

```text
StaffService
```

or:

```text
Appointment
```

Never assume referenced IDs are safe.

---

# 27. Cross-Tenant Reference Prevention

Consider:

```text
Tenant A:
Customer A1

Tenant B:
Staff B1
```

Tenant A must not be able to submit:

```text
CustomerId = A1
StaffId = B1
```

and create a mixed ownership Appointment.

Every related entity participating in a tenant-owned operation must belong to the expected Tenant.

---

# 28. Tenant Stamping

New tenant-owned records should receive TenantId from trusted context.

Possible implementation:

```text
Handler assignment
Domain factory
EF interceptor
SaveChanges interceptor
```

The exact technique may vary.

If automation/interceptors are used:

- behavior must remain understandable,
- tests must prove correct stamping,
- security must not depend only on hidden interceptor behavior.

---

# 29. Database Relationships

Whenever practical, relational design should make invalid ownership relationships difficult.

For example, indexes and foreign keys should support consistent ownership.

However, ordinary foreign keys alone may not prevent all cross-tenant combinations.

Application-level validation remains required.

More advanced composite ownership constraints may be introduced selectively if they provide meaningful protection without excessive complexity.

---

# 30. Database Indexing

Tenant-owned high-volume tables should consider indexes beginning with or including TenantId.

Examples:

```text
(TenantId, Id)
(TenantId, Mobile)
(TenantId, Status)
(TenantId, BranchId, Date)
(TenantId, StaffId, Date)
```

Indexes must match actual query patterns.

Do not create every possible TenantId combination mechanically.

---

# 31. Unique Constraints

Uniqueness should normally be tenant-aware.

Example:

Customer phone uniqueness, if enforced:

Avoid:

```text
UNIQUE(Mobile)
```

Prefer:

```text
UNIQUE(TenantId, Mobile)
```

when the business rule is:

```text
Unique inside business
```

rather than globally across Shinera.

---

# 32. Slug Exception

Some identifiers may need platform-wide uniqueness.

Example:

```text
Tenant.Slug
```

if used in:

```text
/book/{tenantSlug}
```

In such cases global uniqueness may be intentional.

Uniqueness scope must reflect the product rule.

---

# 33. Public Booking

Public booking has no authenticated Tenant context initially.

Tenant is resolved through a public business identifier such as:

```text
tenantSlug
```

Conceptual flow:

```text
/book/{tenantSlug}
↓
Resolve Tenant
↓
Validate Tenant active
↓
Create restricted public tenant context
↓
Execute public booking flow
```

Public context must expose only approved public functionality.

It must not behave like an authenticated Tenant user.

---

# 34. Public Tenant Resolution

Public Tenant resolution must verify:

```text
Tenant exists
Tenant is active
Subscription allows required public feature
Business/branch is publicly available
```

The resolved tenant is then server-controlled.

Client requests cannot switch Tenant arbitrarily after public context resolution.

---

# 35. Background Jobs

Background jobs do not naturally have an HTTP current tenant context.

Every tenant-aware background operation must explicitly establish Tenant context.

Example:

```text
Reminder Job
↓
Appointment TenantId
↓
Create Tenant execution scope
↓
Load appointment
↓
Send reminder
```

Avoid background code that executes unscoped tenant queries.

---

# 36. Scheduled Multi-Tenant Jobs

For jobs processing all tenants:

Preferred conceptual flow:

```text
Resolve active Tenant IDs
↓
For each Tenant
    ↓
Create tenant scope
    ↓
Execute tenant-specific operation
```

Do not run large unfiltered business queries and then hope to separate tenants afterward.

---

# 37. Caching

Tenant-specific cache keys must include Tenant identity.

Bad:

```text
customers:list
```

Good:

```text
tenant:{tenantId}:customers:list
```

Potentially branch-scoped:

```text
tenant:{tenantId}:branch:{branchId}:appointments:{date}
```

Cross-tenant cache leakage is a security defect.

---

# 38. File Storage

Tenant-owned file paths/keys should include Tenant identity.

Conceptual:

```text
tenants/{tenantId}/logos/...
tenants/{tenantId}/staff/...
```

File retrieval must verify access.

Knowing a storage key must not bypass authorization.

---

# 39. Logging

Logs for tenant-aware requests should include, when available:

```text
TenantId
UserId
BranchId
TraceId
```

This improves diagnostics.

Do not log sensitive business or authentication data unnecessarily.

---

# 40. Audit

Audit entries for tenant-owned operations should include:

```text
TenantId
BranchId?
UserId
Action
Entity
EntityId
Timestamp
```

Audit records themselves must respect access control.

---

# 41. Authorization Integration

Tenant membership answers:

```text
Can this user enter this Tenant?
```

Authorization answers:

```text
What can this user do inside the Tenant?
```

Branch access answers:

```text
Where inside the Tenant can they operate?
```

These concerns must remain distinct.

Conceptually:

```text
Authentication
↓
Tenant Membership
↓
Branch Access
↓
Permission
↓
Scope
↓
Feature Entitlement
↓
Operation
```

---

# 42. Subscription Integration

Subscription belongs to Tenant.

Feature availability is therefore tenant-scoped.

Conceptually:

```text
Current Tenant
↓
Subscription
↓
Plan
↓
Features
```

A User moving between Tenants may have different available capabilities.

Frontend must recalculate feature state when workspace Tenant changes.

---

# 43. Dashboard

Dashboard queries must be tenant-scoped.

If Branch filter is selected:

```text
TenantId = currentTenant
AND
BranchId = selected accessible branch
```

Dashboard must never aggregate data across unrelated Tenants for ordinary business users.

Platform analytics is a separate system-level capability.

---

# 44. Tenant Lifecycle

Conceptual Tenant status:

```text
Active
Suspended
Inactive
```

Exact enum may differ.

Inactive/suspended tenant behavior should centrally prevent ordinary operations.

Do not scatter:

```text
if (!tenant.IsActive)
```

through every handler if a centralized mechanism can enforce workspace validity.

Exact mechanism may be refined later.

---

# 45. Tenant Deletion

Hard deletion of Tenant is dangerous because of:

```text
Appointments
Payments
Audit
Subscriptions
Historical records
```

Initial architecture should prefer lifecycle state over immediate destructive deletion.

Possible future process:

```text
Deactivate
↓
Retention period
↓
Export / legal handling
↓
Controlled purge
```

Full Tenant deletion is not an MVP requirement.

---

# 46. Soft Delete Interaction

Soft-deleted tenant-owned records remain owned by the Tenant.

Any filter composition must maintain:

```text
Tenant Filter
AND
Soft Delete Filter
```

Disabling soft-delete filtering must not implicitly disable Tenant isolation.

---

# 47. Migrations

Because all Tenants share the same schema:

```text
One migration
→ applies to all tenants
```

Benefits:

- simple deployment,
- consistent schema,
- no per-tenant migration orchestration.

Migration review must consider existing data from all tenants.

---

# 48. Reporting Across Tenants

Ordinary tenant reports:

```text
Tenant-scoped
```

Platform reports:

```text
Cross-tenant
```

Cross-tenant reporting is a privileged platform concern.

It must not reuse ordinary tenant user endpoints without explicit system authorization.

---

# 49. Support / Impersonation

Future support tooling may require platform operators to inspect a Tenant.

If introduced, it must include:

```text
Explicit operator permission
Explicit target Tenant
Audit trail
Visible impersonation/support state
No hidden privilege escalation
```

This is not part of MVP.

---

# 50. Testing Strategy

Tenant isolation requires dedicated automated tests.

Testing must go beyond checking that a query contains TenantId.

Required behavioral tests should prove actual isolation.

---

# 51. Minimum Tenant Isolation Test Matrix

For a critical resource:

```text
Tenant A Resource A
Tenant B Resource B
```

Test:

```text
A reads A → allowed
A reads B → denied / not found

A updates A → allowed
A updates B → denied / not found

A deletes A → allowed
A deletes B → denied / not found
```

For relationships:

```text
A references A entity → allowed
A references B entity → denied
```

---

# 52. Appointment Isolation Tests

Appointments require tests for mixed tenant IDs.

Example:

```text
Tenant A Customer
Tenant A Service
Tenant B Staff
```

Attempting to create:

```text
Appointment(A Customer, A Service, B Staff)
```

must fail.

Similarly:

```text
Branch
Service
Staff
Customer
Appointment
Payment
```

relationships must remain tenant-consistent.

---

# 53. Security Semantics for Missing Foreign Tenant Resource

When a user requests an entity from another Tenant, prefer behavior that does not reveal its existence.

Often:

```text
Not Found
```

is preferable to:

```text
Forbidden: Resource belongs to another Tenant
```

Exact API behavior should stay consistent across the project.

The important rule is:

```text
Do not leak cross-tenant existence unnecessarily.
```

---

# 54. Repository Enforcement

Backend implementation should include reusable tenant-safe patterns.

Candidates:

```text
ICurrentTenant
Tenant-aware query filters
Tenant ownership validation helpers
Tenant integration test fixtures
```

Avoid creating a generic framework so abstract that ownership becomes difficult to see.

Tenant safety should be explicit enough for code review.

---

# 55. Frontend Responsibilities

Frontend maintains active workspace state for UX.

It may know:

```text
Current Tenant
Current Branch
Available Tenant memberships
Available Branches
```

But frontend state is not trusted by backend.

Changing browser state must not grant access.

---

# 56. Tenant Switching

When Tenant changes, frontend should invalidate tenant-specific state.

Examples:

```text
Dashboard data
Cached lists
Current Branch
Permissions
Feature entitlements
Appointment filters
```

Do not allow stale data from Tenant A to remain visible in Tenant B workspace.

---

# 57. Branch Switching

Branch switching should validate that:

```text
Branch.TenantId == Current Tenant
```

and user has branch access.

Tenant-wide feature data may remain.

Branch-scoped data must reload.

---

# 58. Authentication Token Strategy

Tenant selection should not force architecture into unsafe permanent Tenant assumptions.

Possible approaches include:

```text
Tenant claim in short-lived access token
Server-side workspace/session context
Validated tenant header tied to membership
```

The exact transport mechanism may be refined in the authentication/authorization ADR.

Regardless of implementation:

```text
membership must be validated
```

and arbitrary tenant IDs must not become trusted.

---

# 59. Tenant Context Failure

If tenant context is required but unavailable:

```text
Tenant-scoped operation must fail
```

Do not fall back to:

```text
first tenant
system tenant
empty tenant
all tenants
```

Silent fallback is forbidden.

---

# 60. Performance

Shared-schema tenancy allows efficient querying if TenantId is consistently indexed.

Most operational queries will naturally start with:

```text
TenantId
```

This improves selectivity.

For large tables such as:

```text
Appointments
Payments
AuditLog
```

indexing strategy should include Tenant context.

---

# 61. Scalability

The selected strategy is expected to support Shinera's MVP and substantial SaaS growth.

If future scale creates a need, tenants may later be partitioned by:

```text
Database
Shard
Region
```

The current Tenant abstraction provides a migration path.

Physical tenant partitioning is not required now.

---

# 62. Consequences

## Positive

This decision gives Shinera:

- simple infrastructure,
- simple migrations,
- low operational cost,
- fast tenant provisioning,
- easy reporting,
- consistent domain model,
- simple Solo/Salon evolution,
- straightforward local development.

## Negative

It requires strict discipline around:

- query filtering,
- ownership validation,
- cache keys,
- background jobs,
- system operations,
- test coverage.

A single unsafe query can become a serious security issue.

---

# 63. Rejected Shortcuts

The following are explicitly rejected.

## TenantId From Request as Trust

Rejected:

```text
Request says TenantId X
→ assume user belongs to X
```

## Frontend Filtering Only

Rejected:

```text
Frontend only shows current Tenant records
→ Backend returns unrestricted records
```

## Branch as Tenant

Rejected.

Branches belong to Tenants.

## Separate Solo Data Model

Rejected.

Solo uses Tenant + Main Branch.

## Global Query Filter as Only Protection

Rejected.

Defense in depth is required.

---

# 64. Implementation Guidance

A typical create handler should conceptually resemble:

```text
Get current Tenant
↓
Validate referenced entities belong to Tenant
↓
Create entity with TenantId
↓
Save
```

A typical query:

```text
Current Tenant
↓
Tenant-scoped query
↓
Projection
↓
Response
```

---

# 65. Example — Create Service

Conceptually:

```text
POST /services
{
    name,
    categoryId,
    duration,
    price
}
```

Server:

```text
Current Tenant = T1

Validate Category:
Category.Id == categoryId
AND
Category.TenantId == T1

Create:
Service.TenantId = T1
```

Client does not decide Tenant ownership.

---

# 66. Example — Appointment

Request:

```text
CustomerId
ServiceId
StaffId
BranchId
Date
Time
```

Backend validates:

```text
Branch.TenantId == CurrentTenant
Customer.TenantId == CurrentTenant
Service.TenantId == CurrentTenant
Staff.TenantId == CurrentTenant
```

Then validates branch/staff/service business relationships.

Only then may Appointment be created.

---

# 67. Example — Public Booking

Route:

```text
/book/{tenantSlug}
```

Server:

```text
Resolve slug → Tenant
↓
Create restricted public tenant scope
↓
Return public Branch/Service information
↓
Process booking under resolved Tenant
```

Incoming appointment payload does not choose another Tenant.

---

# 68. Operational Rule

Any code touching tenant-owned data must trigger the review question:

> How is Tenant isolation enforced here?

If the answer is unclear, the implementation is incomplete.

---

# 69. Definition of Done Integration

For tenant-owned stories:

```text
Tenant Isolation = Applicable
```

by default.

A story cannot be marked Done unless isolation is verified.

`N/A` requires a genuine non-tenant reason such as:

```text
pure static UI
platform-global configuration
non-business technical utility
```

---

# 70. Follow-Up ADRs

This decision interacts directly with:

```text
ADR-002 — Authentication with OpenIddict
ADR-003 — Authorization and Permission Model
ADR-004 — Appointment Concurrency Strategy
ADR-006 — Subscription and Feature Gating
ADR-007 — Staff vs User Identity Model
```

Those ADRs should preserve the tenancy model established here.

---

# 71. Final Decision Summary

Shinera will use:

```text
Shared Database
+
Shared Schema
+
Explicit Tenant ownership
+
Trusted CurrentTenant context
+
EF Core tenant filtering
+
Application ownership validation
+
Backend authorization
+
Tenant isolation integration tests
```

Branches are subdivisions of a Tenant, not separate tenants.

Solo businesses use the same Tenant + Main Branch model as salons.

Tenant identity is resolved by the server from trusted context and never trusted solely from client input.

All tenant-owned operations must preserve tenant consistency across every referenced entity.

Tenant isolation is treated as a security boundary and a mandatory Definition of Done requirement.
