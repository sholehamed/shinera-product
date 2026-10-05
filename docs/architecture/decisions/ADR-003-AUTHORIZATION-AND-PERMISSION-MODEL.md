# ADR-003 — Authorization and Permission Model

**Project:** Shinera  
**Status:** Accepted  
**Date:** 2026-10-05  
**Decision Type:** Architecture / Authorization / Security  
**Scope:** Backend, Frontend, Multi-Tenancy, Branch Access, Permissions, Roles, UI Visibility  
**Related Documents:**
- `SHINERA-PRD.md`
- `SHINERA-PRODUCT-BACKLOG.md`
- `PROJECT-INSTRUCTIONS.md`
- `ARCHITECTURE-OVERVIEW.md`
- `DEFINITION-OF-DONE.md`
- `ADR-001-MULTI-TENANCY-STRATEGY.md`
- `ADR-002-AUTHENTICATION-WITH-OPENIDDICT.md`

---

# 1. Context

Shinera requires authorization for several user types:

```text
Owner
Admin
Receptionist
Staff / Specialist
Customer
```

Simple role checks are not sufficient.

A user may:

```text
belong to one or more Tenants,
have access to some Branches,
perform some actions,
see only their own data,
have permissions inherited from Role,
receive direct User permissions,
and operate under Subscription feature constraints.
```

Therefore authorization must answer:

```text
Who is the authenticated User?
Which Tenant is active?
Which Branch is active?
Which Resource is requested?
Which Action is requested?
Which Scope applies?
Does the subject have the Permission?
Does the Subscription allow the Feature?
Does ownership/resource context satisfy the Scope?
```

Authorization is a security boundary.

Frontend visibility is not authorization.

---

# 2. Decision

Shinera will use a dynamic permission model based on:

```text
Resource
+
Action
+
Scope
```

Permissions may be assigned to:

```text
Role
or
User
```

Assignments may apply at:

```text
Tenant
Branch
Own
Child
```

scope levels where supported.

The backend is authoritative.

The frontend consumes permission information only for UX decisions such as:

```text
route visibility
menu visibility
button visibility
disabled actions
```

API resources and UI resources are represented separately.

---

# 3. High-Level Model

Conceptually:

```text
User
 ├── TenantMembership
 │     └── Roles
 │
 ├── Direct PermissionAssignments
 │
 └── BranchMemberships

Role
 └── PermissionAssignments

Permission
 ├── Resource
 ├── Action
 └── Scope
```

Authorization flow:

```text
Authenticated User
      ↓
Current Tenant
      ↓
Branch Access
      ↓
Effective Permission Set
      ↓
Requested Resource + Action
      ↓
Scope Evaluation
      ↓
Feature Entitlement
      ↓
Allow / Deny
```

---

# 4. Why Role-Based Authorization Alone Is Rejected

Pure RBAC such as:

```text
Owner
Admin
Receptionist
Staff
```

is insufficient.

Example:

```text
Staff A may view their own appointments.
Receptionist may view all appointments in Branch A.
Manager may view all appointments in Tenant.
```

All three may need:

```text
appointments.view
```

but with different scope.

Therefore:

```text
Role != Authorization Rule
```

Role is a container for permissions.

---

# 5. Core Authorization Concepts

The model uses the following concepts:

```text
ApiResource
UiResource
Permission
PermissionAssignment
Role
TenantMembership
BranchMembership
Scope
```

Each concept has one clear responsibility.

---

# 6. API Resource

`ApiResource` represents a backend business/security resource.

Examples:

```text
appointments
customers
services
staff
payments
reports
settings
branches
subscriptions
```

The resource name must be:

```text
stable
machine-readable
business-oriented
```

Avoid resource names tied to controller or route implementation details.

Good:

```text
appointments
```

Avoid:

```text
AppointmentController
ApiV1AppointmentEndpoint
```

---

# 7. UI Resource

`UiResource` represents a frontend capability or visible application element.

Examples:

```text
appointments.page
appointments.create.button
customers.page
reports.revenue.widget
staff.schedule.page
settings.subscription.page
```

UI resources exist for UX control.

They are not security boundaries.

A hidden button must never be considered equivalent to backend permission enforcement.

---

# 8. Why API and UI Resources Are Separate

API permission and UI visibility are related but not identical.

Example:

A user may have:

```text
appointments.view
```

API access.

But the UI may separately decide whether to show:

```text
appointments.page
dashboard.appointments.widget
customer.appointment-history
```

Keeping them separate allows:

- one API permission to support multiple UI surfaces,
- UI redesign without changing API authorization semantics,
- clearer backend security rules,
- finer UX feature control.

---

# 9. Action Model

Permissions are action-oriented.

Initial actions may include:

```text
view
create
update
delete
cancel
manage
assign
activate
deactivate
complete
reschedule
export
```

Use specific verbs when they represent distinct business meaning.

Example:

```text
appointments.cancel
appointments.reschedule
appointments.complete
```

is preferable to only:

```text
appointments.update
```

when these operations have different business/security implications.

---

# 10. Permission Key

Permission keys should be stable and human-readable.

Recommended format:

```text
resource.action
```

Examples:

```text
appointments.view
appointments.create
appointments.reschedule
appointments.cancel
appointments.complete

customers.view
customers.create
customers.update

services.view
services.create
services.update
services.delete

staff.view
staff.manage

payments.view
payments.create

reports.view
settings.manage
branches.manage
```

Permission keys become long-lived API/application identifiers.

Do not rename them casually.

---

# 11. Scope Model

Permission alone answers:

```text
What action?
```

Scope answers:

```text
Over which data?
```

Initial scope types:

```text
Tenant
Branch
Own
Child
```

Only scopes actually implemented may be used in production.

---

# 12. Tenant Scope

`Tenant` means the permission applies across the current Tenant.

Example:

```text
appointments.view + Tenant
```

means:

```text
User can view appointments across all allowed Tenant data
```

subject to other constraints such as:

```text
Subscription
resource state
system rules
```

Tenant scope does not grant access to another Tenant.

---

# 13. Branch Scope

`Branch` means the permission applies inside an allowed Branch.

Example:

```text
appointments.view + Branch
```

means:

```text
User can view appointments for the active/authorized Branch
```

The Branch must:

```text
belong to Current Tenant
and
be accessible to User
```

---

# 14. Own Scope

`Own` means the operation is restricted to data owned by or assigned to the current subject.

Examples:

```text
Staff views own appointments
Staff edits own schedule
User views own profile
```

`Own` is not determined only by:

```text
CreatedBy == UserId
```

Ownership is domain-specific.

For Staff appointments, "Own" may mean:

```text
Appointment.Staff.UserId == CurrentUser.Id
```

Ownership evaluation must be explicit for each resource type.

---

# 15. Child Scope

`Child` represents hierarchical access to subordinate subjects/resources.

Potential example:

```text
Branch Manager
→ manages Staff in managed Branches
```

or future organizational hierarchies.

Because Shinera MVP does not yet require a complex organization tree, `Child` is reserved but should not be broadly implemented until a concrete business requirement exists.

Do not introduce speculative hierarchy logic.

---

# 16. Scope Is Not Role

Avoid:

```text
Staff → Own
Receptionist → Branch
Owner → Tenant
```

as hardcoded authorization logic.

Instead define role defaults as permission assignments.

Example:

```text
Staff Role
→ appointments.view + Own

Receptionist Role
→ appointments.view + Branch

Owner Role
→ appointments.view + Tenant
```

This keeps Role configurable.

---

# 17. Role Model

Initial built-in roles:

```text
Owner
Admin
Receptionist
Staff
```

Roles are permission bundles.

Conceptually:

```text
Role
├── Id
├── TenantId? / System-defined marker
├── Name
├── Key
├── IsSystem
└── IsActive
```

Exact persistence may vary.

---

# 18. System Roles vs Tenant Roles

Shinera may maintain built-in system role templates.

Example:

```text
Owner
Admin
Receptionist
Staff
```

A Tenant may later support custom roles.

System roles should have stable semantic keys.

Tenant-specific customization must not mutate global role definitions shared by other Tenants.

---

# 19. Owner Role

Owner is initially the highest business-level role inside a Tenant.

Owner generally receives broad Tenant-scoped permissions.

However:

```text
Owner != Platform Administrator
```

Owner cannot access:

```text
other Tenants
platform internal data
system administration
```

unless separately authorized.

---

# 20. Platform Administrator

Platform-level administration is separate from Tenant authorization.

Conceptually:

```text
Platform Administrator
```

may operate outside normal Tenant context for support/system operations.

This must use:

```text
separate platform permissions
explicit system context
strong audit
```

Do not overload the Tenant `Owner` role for platform administration.

---

# 21. Permission Entity

Conceptually:

```text
Permission
├── Id
├── Key
├── Resource
├── Action
├── Description
└── IsActive
```

Scope may be:

```text
part of Permission definition
or
part of PermissionAssignment
```

Shinera decision:

```text
Scope belongs to PermissionAssignment
```

This allows the same permission to be assigned at different scopes.

Example:

```text
appointments.view
```

can be assigned:

```text
Role A → Tenant
Role B → Branch
Role C → Own
```

---

# 22. PermissionAssignment

Conceptually:

```text
PermissionAssignment
├── Id
├── TenantId
├── PermissionId
├── SubjectType
├── SubjectId
├── ScopeType
├── ScopeReferenceId?
├── Effect
└── ...
```

Subject types:

```text
Role
User
```

Initial effect:

```text
Allow
```

Explicit Deny is not required for MVP.

---

# 23. Why Explicit Deny Is Deferred

A model with both:

```text
Allow
Deny
```

creates conflict-resolution complexity.

Example:

```text
Role allows
User denies
Branch allows
Tenant denies
```

MVP does not need this complexity.

Therefore initial model is:

```text
effective permission exists → allow
otherwise → deny
```

Default:

```text
Deny
```

Explicit deny may be introduced later if a concrete use case requires it.

---

# 24. Default Deny

Authorization follows:

```text
Default Deny
```

Meaning:

```text
No matching effective permission
→ Deny
```

Do not rely on implicit broad access.

---

# 25. User Permission Assignment

A Permission may be assigned directly to a User.

Use cases:

```text
temporary additional access
exceptional responsibility
special manager access
```

Direct User permissions augment Role permissions.

They should not become the primary replacement for Roles.

---

# 26. Role Permission Assignment

Typical authorization comes from:

```text
User
→ Role
→ PermissionAssignment
```

Roles simplify repeated access configurations.

Example:

```text
Receptionist
→ customers.view + Branch
→ customers.create + Branch
→ appointments.view + Branch
→ appointments.create + Branch
→ appointments.reschedule + Branch
```

---

# 27. Effective Permission Set

Effective permissions are computed from:

```text
Role assignments
+
Direct User assignments
```

within:

```text
Current Tenant
```

and relevant Branch context.

Conceptually:

```text
EffectivePermissions(User, Tenant, Branch)
```

must be deterministic.

---

# 28. Tenant Membership

Before permission evaluation:

```text
User must have valid TenantMembership
```

Permission assignment must not bypass Tenant membership.

Flow:

```text
Authenticated User
↓
Tenant Membership?
    no → deny
    yes
↓
Permission evaluation
```

---

# 29. Branch Membership

Branch-scoped access may require BranchMembership.

Flow:

```text
Current Tenant valid
↓
Requested Branch belongs to Tenant
↓
User has access to Branch
↓
Branch-scoped permission evaluation
```

Having a Tenant permission does not automatically mean Branch membership is irrelevant.

Exact behavior depends on the permission scope.

---

# 30. Tenant Scope vs Branch Membership

For Tenant-scoped permission:

```text
appointments.view + Tenant
```

the user may view across Tenant branches if that is the intended role capability.

For Branch-scoped permission:

```text
appointments.view + Branch
```

the user may only view allowed Branches.

This distinction should be explicit.

---

# 31. Resource Ownership

For `Own` scope, each resource defines its ownership resolver.

Examples:

## Appointment

```text
Appointment.Staff.UserId == CurrentUser.Id
```

## User Profile

```text
User.Id == CurrentUser.Id
```

## Staff Schedule

```text
Staff.UserId == CurrentUser.Id
```

Do not implement a universal `CreatedBy` ownership rule.

---

# 32. Authorization Evaluation

Conceptual authorization request:

```text
Authorize(
    user,
    tenant,
    branch,
    resource,
    action,
    resourceContext?)
```

Evaluation:

```text
1. Is User authenticated?
2. Is Tenant valid?
3. Does User belong to Tenant?
4. Is Branch valid if required?
5. Does User have effective permission?
6. Does requested scope match resource context?
7. Does Subscription allow required Feature?
8. Is resource/business state valid?
9. Allow / Deny
```

---

# 33. Feature Gating Order

Authorization and Subscription are separate concerns.

Recommended request flow:

```text
Authentication
↓
Tenant Membership
↓
Branch Access
↓
Permission
↓
Feature Entitlement
↓
Business Rule
```

Exact middleware/handler ordering may vary.

The important point is:

```text
Permission != Feature
```

A user may have permission to use a feature that their Tenant has not purchased.

Result:

```text
Denied by entitlement
```

---

# 34. API Authorization

Backend endpoints must declare authorization requirements in a reusable way.

Preferred conceptual style:

```text
RequirePermission(
    resource: "appointments",
    action: "create"
)
```

or:

```text
RequirePermission(AppointmentPermissions.Create)
```

Avoid repeated manual checks such as:

```text
if (!user.IsOwner && !user.IsAdmin && ...)
```

inside every endpoint.

---

# 35. Dynamic Authorization

Shinera should use dynamic authorization policies/handlers where appropriate.

Conceptually:

```text
PermissionRequirement
PermissionAuthorizationHandler
PermissionPolicyProvider
```

Exact ASP.NET Core implementation may vary.

The goal is:

```text
Endpoint declares requirement
Authorization subsystem evaluates it
```

rather than endpoint-specific duplicated logic.

---

# 36. Endpoint Metadata

Permission requirements should be visible near the endpoint definition.

Example conceptual Minimal API:

```text
MapPost(...)
.RequirePermission(AppointmentPermissions.Create)
```

This improves:

```text
discoverability
reviewability
security auditing
```

---

# 37. Permission Constants

Permission keys should be centrally defined.

Example:

```text
AppointmentPermissions.View
AppointmentPermissions.Create
AppointmentPermissions.Reschedule
AppointmentPermissions.Cancel
```

Avoid scattering raw strings:

```text
"appointments.create"
```

through the codebase.

The database/catalog key remains the stable string representation.

---

# 38. Permission Catalog

Shinera should maintain a known permission catalog.

Initial examples:

```text
Tenancy
- tenants.view
- tenants.update

Branches
- branches.view
- branches.manage

Services
- services.view
- services.create
- services.update
- services.delete

Staff
- staff.view
- staff.create
- staff.update
- staff.manage

Schedules
- schedules.view
- schedules.manage

Customers
- customers.view
- customers.create
- customers.update

Appointments
- appointments.view
- appointments.create
- appointments.reschedule
- appointments.cancel
- appointments.start
- appointments.complete
- appointments.no_show

Payments
- payments.view
- payments.create
- payments.refund

Reports
- reports.view

Settings
- settings.manage

Subscription
- subscription.view
- subscription.manage
```

Catalog evolves with business features.

---

# 39. Permission Naming Rule

Permission keys should:

```text
use lowercase
use stable resource names
use clear action verbs
avoid route/version names
avoid UI wording
```

Good:

```text
appointments.reschedule
```

Bad:

```text
appointmentEditButtonAccess
v1AppointmentPatch
```

---

# 40. Permission Seeding

System permission catalog should be seeded deterministically.

Requirements:

```text
Idempotent
Stable keys
Safe on repeated startup/migration
Does not delete tenant assignments accidentally
```

Adding a new Permission should not reset existing Tenant role configuration.

---

# 41. Role Seeding

Default role templates should be seeded consistently.

A newly created Tenant should receive the required baseline role setup.

Conceptual provisioning:

```text
Create Tenant
↓
Create default roles
↓
Assign default role permissions
↓
Assign Owner role to Owner membership
```

Whether roles are copied per Tenant or linked to immutable templates is an implementation choice to be finalized in code/ADR refinement.

The rule is:

```text
Tenant customization must remain isolated.
```

---

# 42. Owner Bootstrap

Registration provisioning must ensure:

```text
Owner User
↓
TenantMembership
↓
Owner Role
↓
Required Tenant-scoped permissions
```

There must be no state where a successfully provisioned Tenant has no authorized Owner.

---

# 43. Permission Scope Reference

Some scope assignments may require a concrete reference.

Example:

```text
ScopeType = Branch
ScopeReferenceId = BranchId
```

For general Branch permission that applies to all BranchMemberships, the reference may not be needed.

This should be modeled intentionally.

Avoid overcomplicating the first implementation.

---

# 44. Scope Resolution Strategy

MVP preferred strategy:

```text
Tenant Scope
→ applies to Current Tenant

Branch Scope
→ applies to current/authorized Branches

Own Scope
→ evaluated using resource ownership resolver

Child Scope
→ deferred unless concretely required
```

Do not create an arbitrary recursive scope engine before a real Child use case exists.

---

# 45. Authorization Context

Authorization handlers may require contextual information such as:

```text
TenantId
BranchId
ResourceId
StaffId
OwnerUserId
```

When resource-specific scope evaluation is required, authorization may happen:

```text
before loading
and/or
after loading minimal resource metadata
```

Use the smallest safe query necessary.

---

# 46. Resource-Based Authorization

Some permissions cannot be fully evaluated only from endpoint metadata.

Example:

```text
appointments.view + Own
```

requires knowing who owns/serves the Appointment.

Therefore Shinera supports resource-based authorization where needed.

Conceptual:

```text
Load minimal Appointment authorization context
↓
Evaluate Own/Branch/Tenant scope
↓
Proceed
```

---

# 47. Query Authorization

List queries require scope-aware filtering.

Example:

```text
appointments.view + Tenant
→ filter by Tenant

appointments.view + Branch
→ filter by allowed Branches

appointments.view + Own
→ filter by current Staff/User ownership
```

Do not:

```text
load all Tenant appointments
then remove unauthorized rows in memory
```

Authorization must shape the database query where practical.

---

# 48. Command Authorization

Commands require both:

```text
permission check
+
resource ownership/scope validation
```

Example:

```text
appointments.cancel + Branch
```

must validate:

```text
Appointment belongs to Current Tenant
Appointment belongs to accessible Branch
```

before cancellation.

---

# 49. Not Found vs Forbidden

Cross-tenant and unauthorized resource lookups should avoid unnecessary resource existence leakage.

For resource-specific access:

```text
Not Found
```

may be returned when the user should not know the resource exists.

For general action denial where the resource is already legitimately visible:

```text
Forbidden
```

may be appropriate.

The project should remain consistent.

---

# 50. Frontend Permission State

Frontend obtains effective authorization state from trusted backend APIs.

Preferred source:

```text
/auth/me
```

or a dedicated authorization bootstrap endpoint.

Possible payload:

```text
user
tenant
branch
roles
permissions
features
```

Frontend must not construct permissions solely from local role names.

---

# 51. Frontend Permission Service

Angular should expose a centralized permission API.

Conceptual:

```text
permissionService.has("appointments.create")
```

and optionally:

```text
permissionService.hasAny(...)
permissionService.hasAll(...)
```

Do not scatter raw permission-array logic throughout components.

---

# 52. Frontend Permission Directive

A reusable directive may be provided.

Conceptual:

```html
<button *hasPermission="'appointments.create'">
    ایجاد نوبت
</button>
```

or modern Angular equivalent.

The exact syntax should follow Angular 20/project conventions.

---

# 53. Route Authorization

Frontend routes may use permission-aware guards.

Example:

```text
/reports
→ reports.view
```

Route guards improve UX.

They do not replace backend security.

---

# 54. Menu Authorization

Navigation should only show relevant resources.

Example:

```text
No staff.view
→ hide Staff menu
```

If a user directly enters the URL:

```text
Frontend guard may reject
+
Backend still enforces access
```

---

# 55. UI Resource Catalog

UI Resource may support visibility that is not identical to backend Permission.

Examples:

```text
dashboard.revenue.widget
appointments.create.button
settings.subscription.menu
```

This can support:

```text
role-specific UX
plan-specific UX
feature discovery
```

without polluting backend API permission definitions.

---

# 56. UI Resource Assignment

UI visibility may be derived from:

```text
Permission
Feature
UI Resource assignment
```

Avoid making UI Resource assignment mandatory for every component in MVP.

Start with high-value surfaces.

Do not build an excessively granular CMS-style UI permission engine prematurely.

---

# 57. Permission vs UI Feature

Example:

```text
appointments.create
```

is a backend business permission.

```text
appointments.create.button
```

is a UI resource.

The UI button should generally depend on:

```text
appointments.create permission
+
required feature
```

A separate UI Resource is only needed if product configuration requires it.

---

# 58. Caching Effective Permissions

Permission resolution may be cached.

Potential cache key:

```text
tenant:{tenantId}:user:{userId}:permissions
```

If Branch affects effective permission:

```text
tenant:{tenantId}:branch:{branchId}:user:{userId}:permissions
```

Cache must be invalidated when:

```text
Role assignment changes
Permission assignment changes
Branch membership changes
User status changes
Tenant membership changes
```

Stale authorization cache is a security concern.

---

# 59. Permission Cache Strategy

Do not introduce distributed permission caching before needed.

Initial approach may use:

```text
in-memory scoped/request caching
or
short-lived server cache
```

if measurement justifies it.

Correctness is more important than optimization.

---

# 60. Permission Query Performance

Avoid querying the full permission graph repeatedly inside one request.

Preferred:

```text
Resolve authorization context once
↓
Reuse within request
```

Potential request-level cache:

```text
EffectivePermissionSet
```

---

# 61. Role Changes

Role/permission changes should affect subsequent authorization checks without requiring the user to obtain a brand-new long-lived token.

This is why dynamic permissions are not treated as permanent access-token claims.

If caching is used, invalidation must keep changes reasonably fresh.

---

# 62. Tenant Switching

When Tenant changes:

Frontend must refresh:

```text
roles
permissions
branches
features
workspace state
```

Backend evaluates permissions for the selected Tenant.

Permissions from Tenant A must never leak into Tenant B.

---

# 63. Branch Switching

When Branch changes:

Frontend should refresh branch-sensitive authorization state where required.

Backend validates:

```text
Branch belongs to Tenant
User has Branch access
```

Branch switching alone does not create permissions.

---

# 64. Staff vs User

A Staff entity may exist without a login User.

Authorization applies to:

```text
User
```

not directly to an offline Staff record.

When Staff has a linked User:

```text
Staff.UserId
```

may be used to evaluate `Own` scope.

Detailed identity relationship belongs to:

```text
ADR-007 — Staff vs User Identity Model
```

---

# 65. Customer Authorization

Customer-facing authentication/authorization may eventually differ from internal salon authorization.

A Customer should not automatically receive internal Employee permissions.

Potential future authorization space:

```text
customer.appointments.view_own
customer.appointments.cancel_own
customer.profile.update_own
```

Exact customer model is outside the core internal authorization MVP unless public/customer login is implemented.

---

# 66. Feature Entitlement Integration

Permissions answer:

```text
May this User perform this action?
```

Feature entitlement answers:

```text
Has this Tenant purchased/enabled this capability?
```

Both must pass.

Example:

```text
User has branches.manage
Tenant lacks Feature.MultipleBranches
→ cannot create second Branch
```

---

# 67. Subscription Admin Example

User:

```text
subscription.manage + Tenant
```

may manage subscription settings if product rules permit.

But Plan/Feature management at platform level is different from Tenant subscription management.

Do not combine:

```text
Platform plan administration
```

with:

```text
Tenant subscription management
```

under one permission without clear intent.

---

# 68. Permission Checks and Business Rules

Authorization must not absorb ordinary business rules.

Example:

Permission:

```text
appointments.cancel
```

Business rule:

```text
Completed appointment cannot be cancelled
```

Flow:

```text
Permission valid
↓
Business rule still evaluated
```

Authorization answers "may attempt", not "operation is valid".

---

# 69. Authorization and Data Validation

Permission success does not imply referenced data is valid.

Example:

```text
User has appointments.create
```

still must validate:

```text
Customer belongs to Tenant
Staff belongs to Tenant
Service belongs to Tenant
Branch belongs to Tenant
Staff provides Service
```

---

# 70. Permission Error Codes

Suggested error codes:

```text
authorization.permission_required
authorization.scope_denied
authorization.branch_denied
authorization.tenant_denied
```

Feature-related errors:

```text
subscription.feature_unavailable
```

Do not make frontend infer the reason from message text.

---

# 71. Unauthorized vs Forbidden

Use:

```text
401
```

when user is not authenticated.

Use:

```text
403
```

when authenticated but lacks permission for a general operation.

For resource-specific hidden resources, `404` may be used to avoid information leakage.

---

# 72. Audit of Authorization Changes

Security-sensitive configuration changes should be audited.

Examples:

```text
Role created
Role deleted
Permission assigned
Permission removed
User role changed
Branch membership changed
Direct permission assigned
```

Audit record should include:

```text
TenantId
ActorUserId
TargetSubject
Action
Timestamp
```

---

# 73. Protection Against Self-Lockout

Critical tenant administration changes should prevent accidental destruction of all administrative access.

Example:

Do not allow:

```text
remove last Owner-equivalent administrator
```

without an explicit safe replacement process.

Exact Owner preservation rules may be implemented in the membership/role domain.

---

# 74. Owner Invariant

A normal active Tenant should always have at least one active Owner or equivalent high-privilege membership.

This is a business/security invariant.

Tenant deactivation/purge may be an exception.

---

# 75. Privilege Escalation Prevention

A user may not assign permissions/roles beyond their management authority.

Example:

A Receptionist with:

```text
staff.view
```

must not gain the ability to assign:

```text
settings.manage
```

simply because a role-management endpoint exists.

Role/permission administration itself requires dedicated permissions and scope checks.

---

# 76. Permission Management Permissions

Potential administrative permissions:

```text
roles.view
roles.create
roles.update
roles.delete
roles.assign

permissions.view
permissions.assign
```

Exact MVP scope may be smaller.

If custom role management is deferred, these permissions may remain platform/internal only.

---

# 77. MVP Simplification

For MVP:

```text
Default built-in roles
+
Permission-based backend enforcement
+
Scope support for Tenant / Branch / Own
+
Direct User assignments if needed
```

are sufficient.

The following may be deferred:

```text
custom role builder UI
explicit deny
complex Child hierarchy
conditional policy DSL
time-based permissions
field-level permissions
```

Do not overbuild IAM before core salon workflows.

---

# 78. Role Customization Roadmap

Future versions may allow Tenant administrators to:

```text
Create custom role
Rename role
Assign permission set
Limit role to Branch
Assign users
```

This must preserve system-critical invariants.

Custom role UI is not required for first MVP.

---

# 79. Policy DSL Is Rejected

Shinera will not initially create a generic policy expression language such as:

```text
IF user.department == ...
AND appointment.price < ...
THEN ...
```

This is unnecessary complexity for current product needs.

Use explicit business authorization handlers.

---

# 80. Field-Level Authorization

Field-level permissions such as:

```text
customers.view_phone
payments.view_reference
```

are not part of MVP unless a concrete data privacy requirement appears.

Authorization remains primarily resource/action/scope based.

---

# 81. Permission Testing

Authorization must have dedicated automated tests.

Minimum matrix:

```text
User with permission → allowed
User without permission → denied
Wrong Tenant → denied
Wrong Branch → denied
Own resource → allowed
Other user's resource under Own scope → denied
Tenant scope → allowed across valid Tenant data
```

---

# 82. Role Tests

Default role tests should verify expected baseline permission mappings.

Example:

```text
Owner
→ expected Tenant admin capabilities

Receptionist
→ expected booking/customer capabilities

Staff
→ expected Own-scope capabilities
```

Tests protect accidental permission drift.

---

# 83. Authorization Integration Tests

Critical endpoint tests should verify authorization through the real HTTP/application pipeline.

Do not test only the handler in isolation.

Examples:

```text
POST /appointments
GET /customers
PUT /staff/{id}
POST /payments
```

---

# 84. Query Scope Tests

For list endpoints:

Create data for:

```text
Tenant A / Branch A1 / Staff A
Tenant A / Branch A2 / Staff B
Tenant B / Branch B1
```

Then verify:

```text
Tenant scope
Branch scope
Own scope
```

return the correct row sets.

---

# 85. Frontend Authorization Tests

Frontend tests should verify:

```text
Permission present → control visible
Permission absent → control hidden/disabled
Feature absent → control unavailable
Tenant switch → permission state refreshed
Branch switch → relevant state refreshed
```

Frontend tests do not replace backend authorization tests.

---

# 86. Permission Catalog Synchronization

Permission catalog must stay synchronized between:

```text
Backend constants
Database seed/catalog
Frontend permission references
Product documentation
```

Prefer generated/shared contract patterns where practical.

Do not maintain unrelated duplicated string sets manually if avoidable.

---

# 87. API/UI Catalog Synchronization

API permissions and UI resources may be exposed through a backend metadata endpoint when useful.

Potential use:

```text
Angular receives effective permission keys
Angular receives feature keys
```

Frontend does not need the entire internal assignment graph.

---

# 88. Authorization Bootstrap Payload

A typical authenticated bootstrap response may contain:

```json
{
  "user": {},
  "workspace": {
    "tenantId": "...",
    "branchId": "..."
  },
  "roles": [],
  "permissions": [],
  "features": []
}
```

Exact shape may evolve.

Keep the frontend contract compact.

---

# 89. Security Boundary

The true authorization boundary is:

```text
Backend
```

Specifically:

```text
Endpoint / Application Authorization
+
Tenant/Branch ownership validation
+
Scope filtering
```

Frontend authorization exists only to improve experience.

---

# 90. Rejected Alternatives

## Hardcoded Role Checks

Rejected:

```text
if user.Role == Owner
```

for general authorization.

## Frontend-Only Authorization

Rejected.

## Permissions Embedded Permanently in Access Token

Rejected as primary source.

## Explicit Deny Model in MVP

Deferred.

## Complex Policy DSL

Rejected.

## Separate Permission Systems Per Feature

Rejected.

Authorization must remain centralized and consistent.

---

# 91. Consequences

## Positive

This model provides:

```text
Flexible roles
Tenant-aware authorization
Branch-aware authorization
Own-data support
Dynamic permission changes
Frontend UX control
Clear security boundaries
Future custom roles
```

## Negative

It introduces complexity around:

```text
Scope evaluation
Permission caching
Role seeding
Resource ownership
Branch access
Testing matrix
```

This complexity is accepted because it directly reflects Shinera's business needs.

---

# 92. Initial Default Role Intent

Initial high-level intent:

## Owner

```text
Tenant-wide administration
Business settings
Services
Staff
Customers
Appointments
Payments
Reports
Subscription
Branches where plan permits
```

## Admin

```text
Broad operational management
Potentially Tenant or Branch scoped
No automatic platform/subscription ownership rights
```

## Receptionist

```text
Customer management
Appointment management
Operational payment actions
Primarily Branch scoped
```

## Staff

```text
Own appointments
Own schedule
Limited customer data required for assigned work
Primarily Own scope
```

Exact permission matrix should be defined separately and may evolve with product stories.

---

# 93. Initial Permission Matrix Principle

Do not freeze every default role permission in this ADR.

This ADR defines the authorization model.

Concrete role-permission mapping belongs in:

```text
Permission Catalog / Seed Configuration
```

and may be adjusted without changing architecture.

Only architectural semantics such as:

```text
Role is a permission bundle
Scope is assignment-level
Default deny
Backend authoritative
```

are fixed here.

---

# 94. Endpoint Example — Create Appointment

Requirement:

```text
appointments.create
```

Flow:

```text
Authenticate
↓
Resolve Tenant
↓
Validate Branch
↓
Resolve effective permissions
↓
Check appointments.create
↓
Evaluate scope
↓
Check required feature
↓
Validate Customer/Staff/Service ownership
↓
Execute command
```

---

# 95. Endpoint Example — View Appointments

User A:

```text
appointments.view + Own
```

Query:

```text
WHERE TenantId = CurrentTenant
AND Staff.UserId = CurrentUser
```

User B:

```text
appointments.view + Branch
```

Query:

```text
WHERE TenantId = CurrentTenant
AND BranchId IN AllowedBranches
```

User C:

```text
appointments.view + Tenant
```

Query:

```text
WHERE TenantId = CurrentTenant
```

This illustrates why authorization must influence query construction.

---

# 96. Endpoint Example — Update Service

Requirement:

```text
services.update
```

Backend:

```text
Resolve current Tenant
↓
Load Service inside Tenant scope
↓
Evaluate permission scope
↓
Apply business validation
↓
Update
```

A Service ID from another Tenant must not be updateable.

---

# 97. Permission Change Propagation

After a permission/role assignment change:

Backend:

```text
new requests should use updated authorization state
```

Frontend:

```text
refresh effective permissions
```

Possible triggers:

```text
workspace bootstrap refresh
explicit permission reload
session event
short cache expiry
```

Do not require logout/login as the normal permission refresh mechanism.

---

# 98. Revocation

When TenantMembership is revoked:

```text
User loses Tenant entry/access
```

even if they still have a valid authentication token.

When BranchMembership is revoked:

```text
Branch-scoped access disappears
```

Authentication remains separate.

---

# 99. Deactivated User

If User becomes inactive:

```text
authentication refresh should fail
+
authorization should deny protected operations
```

according to ADR-002.

Authorization should not continue merely because old permission cache exists.

---

# 100. Deactivated Role / Permission

Inactive role/permission entries should not produce effective access.

Catalog deactivation must be handled carefully to avoid inconsistent state.

System permissions should normally be versioned/evolved rather than casually disabled.

---

# 101. Database Considerations

Likely entities:

```text
Role
Permission
PermissionAssignment
TenantMembership
BranchMembership
ApiResource
UiResource
```

Relevant indexes may include:

```text
Permission.Key
Role(TenantId, Key)
PermissionAssignment(TenantId, SubjectType, SubjectId)
TenantMembership(TenantId, UserId)
BranchMembership(BranchId, UserId)
```

Exact schema belongs to implementation.

---

# 102. Tenant Isolation

All Tenant-specific role and assignment records must include Tenant context where applicable.

A Role or PermissionAssignment belonging to Tenant A must never grant access inside Tenant B.

System/global permission catalog definitions may be shared.

Assignments are tenant-contextual.

---

# 103. System Permission Catalog vs Tenant Assignments

Preferred conceptual separation:

```text
Global Permission Definition
+
Tenant-specific Assignment
```

Example:

```text
Permission:
appointments.create

Tenant A:
Receptionist → Branch

Tenant B:
Receptionist → Tenant
```

This avoids duplicating permission definitions per Tenant.

---

# 104. Role Definition Scope

System default role templates may be global.

Actual tenant role configuration may be:

```text
Tenant-specific role instances
```

to allow future customization.

Exact implementation should favor:

```text
safe tenant isolation
easy seeding
future custom roles
```

---

# 105. Authorization Service Boundary

The application should expose an authorization abstraction.

Conceptual:

```text
IAuthorizationService
IPermissionChecker
ICurrentAuthorizationContext
```

Do not create multiple competing abstractions.

Use ASP.NET Core Authorization integration where it provides value.

The final names should follow existing repository conventions.

---

# 106. Permission Checker Responsibilities

A permission checker may answer:

```text
Does current User have permission key X?
What scope applies?
Which Branches are allowed?
```

It must not become a generic business rules engine.

---

# 107. Authorization Context Example

Conceptual:

```text
CurrentAuthorizationContext
├── UserId
├── TenantId
├── BranchId?
├── Roles
├── EffectivePermissions
├── AllowedBranchIds
└── ...
```

This may be request-scoped and cached for the request lifetime.

---

# 108. Auditability

Authorization decisions should be diagnosable.

For denied operations, logs may include:

```text
UserId
TenantId
BranchId
PermissionKey
Reason code
TraceId
```

Do not log sensitive resource payloads merely for authorization troubleshooting.

---

# 109. Operational Diagnostics

Useful denial reason categories:

```text
NoTenantMembership
NoBranchAccess
MissingPermission
ScopeMismatch
FeatureUnavailable
UserInactive
TenantInactive
```

These may be logged internally.

User-facing responses should avoid revealing sensitive internal structure unnecessarily.

---

# 110. Definition of Done Integration

Any protected Story must explicitly evaluate:

```text
Required Permission
Required Scope
Tenant isolation
Branch access
Feature entitlement
Unauthorized behavior
Frontend visibility
Automated authorization tests
```

If authorization is applicable but unverified:

```text
Story != Done
```

---

# 111. Follow-Up Work

This ADR should be followed by implementation/design work for:

```text
Permission catalog
Default role matrix
Permission seeding
Dynamic authorization handler
Scope evaluator
Frontend permission service
Authorization integration tests
```

Potential supporting document:

```text
PERMISSION-CATALOG.md
```

if the permission list grows substantially.

---

# 112. Follow-Up ADRs

This ADR directly interacts with:

```text
ADR-001 — Multi-Tenancy Strategy
ADR-002 — Authentication with OpenIddict
ADR-006 — Subscription and Feature Gating
ADR-007 — Staff vs User Identity Model
```

Potential future ADR:

```text
ADR-010 — Custom Role Management
```

only when Tenant-defined roles become an active product requirement.

---

# 113. Final Decision Summary

Shinera authorization will use:

```text
Dynamic Permission Authorization
+
Resource.Action permission keys
+
Assignment-level Scope
+
Role-based permission bundles
+
Optional direct User permissions
+
Tenant/Branch membership validation
+
Resource-based Own-scope evaluation
+
Backend enforcement
+
Frontend UX reflection
```

The initial supported scopes are:

```text
Tenant
Branch
Own
```

`Child` is reserved for future concrete hierarchy requirements.

The initial model uses:

```text
Default Deny
```

and does not require explicit deny rules.

API and UI resource catalogs remain separate.

Authentication proves identity.

Authorization proves whether that identity may perform a business action in the current Tenant/Branch/resource context.

Subscription entitlement is evaluated separately from permission.

No frontend visibility rule, role name, token claim, or client-provided Tenant/Branch value is sufficient by itself to authorize a protected backend operation.
