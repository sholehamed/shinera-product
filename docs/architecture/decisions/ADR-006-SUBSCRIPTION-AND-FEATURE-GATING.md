# ADR-006 — Subscription and Feature Gating

**Project:** Shinera  
**Status:** Accepted  
**Date:** 2026-10-05  
**Decision Type:** Architecture / Subscription / Entitlements / Product Access  
**Scope:** Backend, Frontend, Tenant, Plans, Subscription, Authorization, Billing Integration  
**Related Documents:**
- `SHINERA-PRD.md`
- `SHINERA-PRODUCT-BACKLOG.md`
- `PROJECT-INSTRUCTIONS.md`
- `ARCHITECTURE-OVERVIEW.md`
- `DEFINITION-OF-DONE.md`
- `ADR-001-MULTI-TENANCY-STRATEGY.md`
- `ADR-003-AUTHORIZATION-AND-PERMISSION-MODEL.md`
- `ADR-005-DATE-AND-TIME-STRATEGY.md`

---

# 1. Context

Shinera has multiple commercial plans.

Initial product plans:

```text
Solo
Solo Pro
Salon
Salon Pro
```

Different plans may enable different capabilities such as:

```text
Multiple Branches
Advanced Reports
Public Booking
Advanced Notifications
Force Appointment
Higher Staff Limits
Higher Branch Limits
Other future premium capabilities
```

A naive implementation would spread plan checks throughout the codebase:

```csharp
if (tenant.Plan == "SalonPro")
{
    ...
}
```

or in Angular:

```ts
if (plan === 'salon-pro') {
   ...
}
```

This creates tight coupling between:

```text
business capability
and
commercial plan name
```

and makes future pricing/package changes dangerous.

Shinera therefore requires a dedicated entitlement architecture.

---

# 2. Decision

Shinera will use:

```text
Subscription
→ Plan
→ PlanFeature / Entitlement Definition
→ Feature
```

and application code will evaluate:

```text
Feature Capability
```

rather than:

```text
Plan Name
```

The backend is the authoritative source of entitlement.

Frontend feature gating exists only for UX.

---

# 3. Core Rule

Application code asks:

```text
Does Current Tenant have Feature X?
```

not:

```text
Is Current Tenant on Plan Y?
```

Preferred:

```text
Feature.MultipleBranches
```

Rejected:

```text
plan == "SalonPro"
```

---

# 4. High-Level Model

Conceptually:

```text
Tenant
  │
  ▼
Subscription
  │
  ▼
Plan
  │
  ▼
PlanFeature
  │
  ▼
Feature
```

Runtime:

```text
Current Tenant
↓
Resolve effective Subscription
↓
Resolve effective Entitlements
↓
Check Feature / Limit
↓
Allow / Deny
```

---

# 5. Core Entities

Initial subscription architecture contains:

```text
Plan
Feature
PlanFeature
Subscription
```

Optional supporting concepts may be added later, such as:

```text
PlanVersion
SubscriptionOverride
UsageCounter
BillingProviderReference
```

only when required.

---

# 6. Plan

`Plan` represents a commercial package.

Conceptually:

```text
Plan
├── Id
├── Key
├── Name
├── Description
├── IsActive
├── DisplayOrder
└── ...
```

Example stable keys:

```text
solo
solo-pro
salon
salon-pro
```

Plan key is commercial/catalog identity.

It is not used directly for application authorization.

---

# 7. Feature

`Feature` represents a product capability.

Conceptually:

```text
Feature
├── Id
├── Key
├── Name
├── Description
├── Kind
├── IsActive
└── ...
```

Example keys:

```text
multiple_branches
advanced_reports
public_booking
force_appointment
advanced_notifications
```

Feature keys must be:

```text
stable
machine-readable
business-oriented
```

---

# 8. Feature Naming

Preferred keys:

```text
multiple_branches
advanced_reports
public_booking
```

Avoid keys tied to commercial packaging:

```text
salon_pro_feature
gold_plan_reports
premium_branch_mode
```

because capabilities may move between Plans later.

---

# 9. Feature Types

Shinera must support at least two entitlement styles:

```text
Boolean Capability
Numeric Limit
```

Examples:

Boolean:

```text
multiple_branches = enabled
advanced_reports = enabled
```

Limit:

```text
max_branches = 5
max_staff = 25
max_users = 20
```

This avoids encoding product limits as arbitrary code branches.

---

# 10. Feature Kind

Conceptually:

```text
FeatureKind
├── Boolean
└── Limit
```

More complex value types are deferred.

Do not build a generic configuration language inside the Feature system.

---

# 11. Boolean Feature

Example:

```text
Feature:
public_booking

PlanFeature:
Enabled = true
```

Runtime:

```text
entitlements.Has("public_booking")
```

---

# 12. Limit Feature

Example:

```text
Feature:
max_branches

PlanFeature:
Limit = 3
```

Runtime:

```text
entitlements.GetLimit("max_branches")
```

Then:

```text
CurrentBranchCount < MaxBranches
```

must hold before creating another Branch.

---

# 13. Why Limits Are First-Class

Without first-class limits, code tends to become:

```csharp
if (plan == Solo) maxBranches = 1;
if (plan == Salon) maxBranches = 3;
if (plan == SalonPro) maxBranches = 10;
```

This is rejected.

Limits belong to subscription entitlement data.

---

# 14. PlanFeature

`PlanFeature` connects commercial Plans to product Features.

Conceptually:

```text
PlanFeature
├── PlanId
├── FeatureId
├── IsEnabled
├── LimitValue?
└── ...
```

For Boolean features:

```text
IsEnabled
```

is used.

For Limit features:

```text
LimitValue
```

is used.

Exact schema may separate these values if that improves type safety.

---

# 15. Subscription

Each normal Tenant has an effective Subscription.

Conceptually:

```text
Subscription
├── Id
├── TenantId
├── PlanId
├── Status
├── StartedAtUtc
├── EndsAtUtc?
├── CancelledAtUtc?
├── ExternalReference?
└── ...
```

Subscription is Tenant-scoped.

---

# 16. One Effective Subscription

For MVP, a Tenant has:

```text
one effective Subscription
```

at a time.

Historical Subscription records may exist.

The application must resolve one current/effective commercial state.

Do not let multiple overlapping active subscriptions produce ambiguous entitlements.

---

# 17. Subscription Lifecycle

The architecture recognizes that Subscription lifecycle and payment-provider lifecycle are not the same thing.

The application needs to answer:

```text
Is this Subscription currently entitled?
```

Initial domain statuses may include:

```text
Pending
Active
Suspended
Cancelled
Expired
```

Exact commercial semantics such as:

```text
Trial
Past Due
Grace Period
Retrying Payment
```

are product/billing policies and may be added later.

---

# 18. Entitlement-Active Rule

Only an entitlement-eligible Subscription grants paid Features.

Conceptually:

```text
Subscription.IsEntitled(nowUtc)
```

The exact status/time rules must be centralized.

Do not scatter:

```text
subscription.Status == Active
```

through unrelated handlers.

---

# 19. Cancellation Semantics

Cancellation may mean either:

```text
cancel immediately
```

or:

```text
cancel at period end
```

depending on product policy.

This ADR does not define pricing/billing policy.

Architecture requirement:

```text
entitlement resolver
```

must use the Subscription's effective dates/status and expose only the resulting capability state.

---

# 20. Subscription Time

Subscription timestamps follow ADR-005.

Examples:

```text
StartedAtUtc
EndsAtUtc
CancelledAtUtc
```

use UTC instants.

If future commercial policy says:

```text
access until end of local business day
```

that rule must explicitly resolve the configured business timezone.

---

# 21. Tenant Ownership

Subscription belongs to:

```text
Tenant
```

not User.

Therefore:

```text
all Users inside the same Tenant
```

share the Tenant's commercial capabilities.

Permissions still determine which User may operate a Feature.

---

# 22. Permission vs Entitlement

These are separate questions.

Permission:

```text
May this User perform the operation?
```

Entitlement:

```text
Has this Tenant purchased/enabled the capability?
```

Both may be required.

Example:

```text
User has branches.manage
Tenant lacks multiple_branches
```

Result:

```text
User cannot create additional Branch
```

---

# 23. Authorization Flow

Typical protected premium operation:

```text
Authentication
↓
Tenant Membership
↓
Branch Access
↓
Permission Check
↓
Feature Entitlement Check
↓
Usage / Limit Check
↓
Business Validation
↓
Operation
```

Authorization and feature gating should remain conceptually separate even if integrated through endpoint middleware/handlers.

---

# 24. Backend Is Authoritative

Frontend may:

```text
hide
disable
show upgrade CTA
```

for unavailable Features.

But backend must independently enforce entitlement.

A user must not gain a paid Feature by:

```text
editing JavaScript
calling API manually
modifying browser state
```

---

# 25. Frontend Gating

Angular receives effective entitlement data from backend.

Conceptual API:

```text
/auth/me
```

or:

```text
/workspace/bootstrap
```

may include:

```text
features
limits
subscription summary
```

Frontend should not calculate entitlement from plan name.

---

# 26. Frontend Feature Service

Angular should expose a centralized abstraction.

Conceptual:

```ts
featureService.has('multiple_branches')
featureService.limit('max_branches')
```

Do not scatter:

```ts
currentPlan === 'salon-pro'
```

through components.

---

# 27. Frontend Feature Directive

A reusable UI capability may support:

```text
show when feature exists
show upgrade state when absent
```

Conceptual:

```html
<ng-container *hasFeature="'advanced_reports'">
    ...
</ng-container>
```

Exact Angular syntax follows project conventions.

---

# 28. Upgrade UX

Feature gating should support meaningful UX.

Instead of simply hiding every unavailable capability, some surfaces may show:

```text
Locked Feature
Upgrade CTA
Plan comparison
```

This is a product/UX choice.

Security remains backend-enforced regardless.

---

# 29. Feature Constants

Feature keys should be centrally defined.

Conceptual:

```text
Features.MultipleBranches
Features.AdvancedReports
Features.PublicBooking
```

Avoid raw string duplication across backend code.

Stable serialized/database value remains the Feature key.

---

# 30. Limit Constants

Numeric limits should also use stable keys.

Conceptual:

```text
Limits.MaxBranches
Limits.MaxStaff
Limits.MaxUsers
```

These may be represented by the same Feature catalog with `FeatureKind.Limit`.

---

# 31. Entitlement Resolver

Application layer should expose a centralized capability resolver.

Conceptual:

```text
IEntitlementService
```

Responsibilities:

```text
HasFeature(featureKey)
GetLimit(limitKey)
GetEffectiveEntitlements()
```

Exact naming follows repository conventions.

---

# 32. Current Tenant

Entitlement checks resolve using:

```text
ICurrentTenant.TenantId
```

according to ADR-001.

Client-supplied TenantId is not entitlement authority.

---

# 33. Example Entitlement API

Conceptual:

```csharp
await entitlementService.HasFeatureAsync(
    FeatureKeys.MultipleBranches,
    cancellationToken);
```

Limit:

```csharp
var maxBranches =
    await entitlementService.GetLimitAsync(
        FeatureKeys.MaxBranches,
        cancellationToken);
```

Exact API may return a richer result.

---

# 34. Entitlement Result

A richer resolver may expose:

```text
Entitlement
├── FeatureKey
├── Enabled
├── Limit?
└── Source
```

`Source` may be useful for diagnostics:

```text
Plan
Override
System
```

Overrides are not required for MVP.

---

# 35. Feature Enforcement Near Business Operation

Feature checks should exist near the operation they protect.

Example:

```text
Create Branch Command
↓
Check branches.manage permission
↓
Check multiple_branches / max_branches
↓
Create Branch
```

Do not rely only on a global page guard.

---

# 36. Feature Middleware vs Handler

Some Features may be enforced at endpoint metadata/policy level.

Conceptually:

```text
.RequireFeature(FeatureKeys.AdvancedReports)
```

But business-limit checks often belong inside the command/application flow.

Example:

```text
MaxBranches
```

requires counting current Branches and evaluating the operation atomically enough to avoid limit races.

Use the layer that matches the rule.

---

# 37. Boolean Feature Enforcement

Boolean capabilities are good candidates for reusable feature requirements.

Example:

```text
advanced_reports
```

Flow:

```text
endpoint/handler
↓
entitlement resolver
↓
enabled?
```

---

# 38. Numeric Limit Enforcement

Numeric limits need both entitlement and current usage.

Example:

```text
max_branches = 3
```

Create Branch:

```text
Current Branch Count = 3
↓
requested create
↓
deny
```

Suggested error:

```text
subscription.limit_reached
```

---

# 39. Limit Race Conditions

Usage limits must consider concurrency.

Example:

```text
max_branches = 3
current = 2

Request A creates Branch
Request B creates Branch
```

A naive count-then-insert may create 4 Branches.

Critical resource limits should be enforced using appropriate transaction/concurrency protection.

Exact strategy may vary by resource.

Do not assume feature gating is purely read-only.

---

# 40. Main Branch Exception

Every Tenant requires a Main Branch according to ADR-001.

Therefore:

```text
Solo plan
```

may have:

```text
max_branches = 1
```

rather than:

```text
multiple_branches = false
```

as the only rule.

This better represents:

```text
one Branch allowed
```

while premium Plans may increase the limit.

---

# 41. Boolean + Limit Combination

A Feature may have both a capability and limit concept if product UX benefits.

Example:

```text
multiple_branches = true
max_branches = 5
```

However, avoid redundant modeling where:

```text
max_branches > 1
```

alone is sufficient.

Prefer the simplest entitlement representation that expresses product rules clearly.

---

# 42. Initial Recommendation for Branches

Recommended initial model:

```text
max_branches
```

as a numeric entitlement.

Examples:

```text
Solo      → 1
Solo Pro  → 1
Salon     → N
Salon Pro → higher N
```

Exact commercial values belong to product/pricing configuration, not this ADR.

---

# 43. Staff Limits

If product Plans later limit Staff:

```text
max_staff
```

should be a numeric entitlement.

Do not encode:

```text
Solo means exactly one Staff
```

through plan-name conditionals.

---

# 44. Usage Counters

For low-volume resources, usage can be queried directly.

Examples:

```text
Branch count
Staff count
User count
```

Dedicated UsageCounter storage is not required initially.

Introduce counters only when:

```text
query cost
billing usage
high-frequency metering
```

justify them.

---

# 45. Metered Features

Usage-based billing such as:

```text
SMS count
AI credits
booking volume
storage
```

is not part of the initial entitlement architecture.

If introduced, it requires a dedicated metering/billing design.

Do not overload simple `PlanFeature.LimitValue` as a full billing ledger.

---

# 46. Plan Catalog

Plan catalog contains commercial presentation data such as:

```text
Name
Description
Price
Billing Period
Display Order
Marketing copy
```

and entitlement mapping.

Core business code should not depend on marketing text.

---

# 47. Pricing

Pricing is commercial data.

Feature gating does not calculate prices.

A Plan may contain or reference:

```text
price
currency
billing interval
```

but entitlement resolution only cares about effective capability.

---

# 48. Currency

Commercial pricing currency is separate from salon business transaction currency.

Do not assume:

```text
subscription currency
=
customer payment currency
```

unless product policy explicitly says so.

---

# 49. Plan Activation

Inactive Plans:

```text
must not be selectable for new Subscription creation
```

but existing subscriptions may require grandfathered behavior.

Do not automatically disable existing Tenant capabilities merely because:

```text
Plan.IsActive = false
```

if `IsActive` only means:

```text
available for sale
```

These semantics must remain explicit.

---

# 50. Plan Modification Risk

Changing PlanFeature configuration can alter capability for every Tenant referencing that Plan.

This can be intentional or dangerous.

Examples:

```text
Add advanced_reports to Salon
Remove public_booking from Solo Pro
Change max_staff from 10 to 5
```

The architecture must support safe future commercial changes.

---

# 51. Plan Versioning

Shinera should support plan versioning when commercial configuration begins changing for existing customers.

Conceptually:

```text
Plan
└── PlanVersion
      └── PlanFeature
```

A Subscription references the effective PlanVersion.

This enables:

```text
new customers → new version
existing customers → grandfathered version
```

without rewriting code.

---

# 52. MVP Plan Versioning Rule

Plan versioning is recommended in the domain model if inexpensive, but a full pricing/version management UI is not required for MVP.

If PlanFeature rows are edited globally without versioning, the team must understand that existing subscribers change immediately.

Before production paid subscriptions launch, this behavior must be explicitly chosen.

---

# 53. Grandfathering

Grandfathering is a commercial policy, not an authorization concept.

Architecture supports it through:

```text
Subscription → specific PlanVersion
```

if needed.

Do not implement special code such as:

```text
if subscription.CreatedAt < X
```

throughout feature checks.

---

# 54. Trial

Trial behavior is deferred until product rules are finalized.

Possible architecture:

```text
Subscription.Status = Trial
+
trial plan/version
or
trial entitlement overlay
```

Do not hardcode:

```text
if trial then all features
```

without explicit product approval.

---

# 55. Free / Internal Subscription

Every normal Tenant should have a resolvable commercial state.

If free access exists:

```text
Free Plan
or
explicit free entitlement set
```

is preferable to:

```text
Subscription == null means everything works
```

Missing Subscription should fail safely.

---

# 56. Missing Subscription

If a normal Tenant has no resolvable Subscription:

```text
paid/gated capabilities should default deny
```

and workspace may show a subscription/setup state.

Do not silently grant premium capabilities.

---

# 57. System Tenant

The System Tenant from ADR-001 is not governed by ordinary commercial subscription rules unless explicitly required.

System/platform operations use explicit platform context.

Do not create a fake premium Subscription solely to bypass system authorization.

---

# 58. Subscription Provisioning

Registration/workspace provisioning should create or assign the initial Subscription as part of the provisioning transaction where required.

Conceptual:

```text
Create User
↓
Create Tenant
↓
Create Main Branch
↓
Create Membership
↓
Create Subscription
↓
Assign Owner
↓
Commit
```

A successfully registered Tenant should not accidentally enter a state with undefined entitlements.

---

# 59. Plan Selection

Registration may receive a public Plan key:

```text
solo
solo-pro
salon
salon-pro
```

Backend validates:

```text
Plan exists
Plan is available for registration
Plan is active/sellable
commercial requirements satisfied
```

Client-provided pricing or Feature lists are ignored.

---

# 60. Never Trust Client Feature List

Rejected request:

```json
{
  "plan": "salon",
  "features": [
    "advanced_reports",
    "multiple_branches"
  ]
}
```

Backend resolves Feature entitlement from trusted plan/subscription catalog.

---

# 61. Payment Provider Separation

Subscription entitlement must not be tightly coupled to a specific payment provider.

Conceptually:

```text
Billing Provider
↓
Payment/Webhook Result
↓
Subscription domain transition
↓
Entitlement changes
```

Application code does not ask:

```text
Did Stripe/Zarinpal/etc say X?
```

It asks:

```text
What is the current Subscription entitlement?
```

---

# 62. External Billing Reference

Subscription may contain:

```text
ExternalCustomerId
ExternalSubscriptionId
BillingProvider
```

or equivalent integration metadata.

These are infrastructure/integration concerns.

They are not Feature keys.

---

# 63. Billing Webhooks

Future billing provider webhooks must:

```text
verify authenticity
be idempotent
map provider events
update Subscription transactionally
invalidate entitlement cache
audit significant changes
```

Exact provider workflow is outside this ADR.

---

# 64. Immediate Feature Revocation

When Subscription becomes non-entitled:

```text
new protected operations
```

must stop being authorized after entitlement state refresh/invalidation.

Existing business records are not deleted.

Example:

```text
Tenant loses Advanced Reports
→ historical report data remains
→ report access denied
```

---

# 65. Downgrade Principle

Downgrade must not destroy customer data merely because the new Plan has lower limits.

Example:

```text
Tenant has 4 Branches
downgrades to max_branches = 1
```

Do not automatically delete 3 Branches.

Instead enter an over-limit state.

---

# 66. Over-Limit State

When current usage exceeds the new Plan limit:

```text
existing data remains
new growth operations are blocked
```

Example:

```text
Current Branches = 4
MaxBranches = 1

View existing Branches → allowed according to permission
Create new Branch → denied
```

Product policy may additionally restrict editing/activation.

Do not silently destroy data.

---

# 67. Downgrade Remediation

UI should surface:

```text
current usage
allowed limit
required remediation
upgrade option
```

Exact user workflow belongs to product design.

Architecture only guarantees:

```text
data-preserving enforcement
```

by default.

---

# 68. Feature Removal

If a Tenant loses a Boolean Feature:

```text
existing data created by that Feature remains
```

unless explicit product/data-retention rules say otherwise.

Example:

```text
Advanced Reports
```

simply becomes unavailable.

For structural features such as multiple Branches, use over-limit behavior instead of deletion.

---

# 69. Feature Dependencies

Some Features may depend on others.

Example future possibility:

```text
advanced_force_appointment
requires
appointments
```

Do not create a generic recursive Feature dependency engine for MVP.

Plan catalog validation should prevent obviously invalid configurations.

Introduce dependency metadata only when real needs arise.

---

# 70. Feature Gating and Routes

Backend routes may exist regardless of Tenant Plan.

Entitlement determines whether the current Tenant may use them.

Do not generate entirely different API deployments per Plan.

---

# 71. Feature Gating and Database

All Tenants share one schema.

Premium Feature data may exist in shared tables.

Subscription does not change database schema per Tenant.

This preserves:

```text
shared-schema multi-tenancy
```

from ADR-001.

---

# 72. Feature Gating and Migrations

Database migrations apply globally.

A disabled Feature does not mean its schema is absent for a Tenant.

Feature gating occurs at runtime.

---

# 73. Feature Gating and Background Jobs

Background work must also respect entitlement where business behavior requires it.

Example:

```text
Premium reminder campaign
```

should not continue indefinitely after Feature revocation unless product policy explicitly allows already-scheduled work.

Jobs establish Tenant context and resolve current entitlement.

---

# 74. Feature Gating and Notifications

A core operational notification may be required for system correctness regardless of premium notification features.

Do not conflate:

```text
core transactional notification
```

with:

```text
premium notification automation
```

Feature keys should represent meaningful product capabilities.

---

# 75. Feature Gating and Public Booking

Public Booking request:

```text
resolve Tenant
↓
check Tenant active
↓
check public_booking entitlement
↓
apply public booking rules
↓
allow/deny
```

A public route existing in the application does not mean every Tenant has public booking enabled.

---

# 76. Feature Gating and Dashboard

Dashboard widgets may be premium.

Frontend:

```text
hide/lock widget
```

Backend:

```text
premium query endpoint or data section
→ enforce entitlement
```

Do not return premium data and rely only on UI hiding it.

---

# 77. Feature Gating and Authorization

A reusable operation may require both:

```text
Permission Key
Feature Key
```

Example:

```text
Permission:
reports.view

Feature:
advanced_reports
```

User with one but not the other cannot execute the premium report.

---

# 78. Entitlement Error Codes

Suggested errors:

```text
subscription.required
subscription.inactive
subscription.feature_unavailable
subscription.limit_reached
subscription.over_limit
```

Frontend behavior must depend on stable codes, not message parsing.

---

# 79. HTTP Semantics

For authenticated, authorized User whose Tenant lacks a commercial capability:

Preferred API behavior may use:

```text
403 Forbidden
```

with:

```text
subscription.feature_unavailable
```

because the caller is not entitled to the operation.

For resource creation blocked by limit:

```text
409 Conflict
```

or:

```text
403 Forbidden
```

may be used depending on established API conventions.

The important contract is stable error code.

---

# 80. Entitlement Bootstrap

Workspace bootstrap may return:

```json
{
  "subscription": {
    "planKey": "salon-pro",
    "status": "active"
  },
  "features": [
    "advanced_reports",
    "public_booking"
  ],
  "limits": {
    "max_branches": 5,
    "max_staff": 50
  }
}
```

Exact payload may evolve.

Frontend should not need the full PlanFeature database graph.

---

# 81. Plan Name in Frontend

Frontend may display:

```text
Salon Pro
```

for commercial UX.

But operational logic uses:

```text
feature keys / limits
```

Plan key may be used for:

```text
pricing page
billing display
upgrade comparison
```

not authorization.

---

# 82. Entitlement Cache

Effective entitlements are good cache candidates because:

```text
read frequently
change infrequently
```

Potential cache key:

```text
tenant:{tenantId}:entitlements
```

---

# 83. Cache Invalidation

Invalidate entitlement cache when:

```text
Subscription changes
PlanVersion changes
PlanFeature changes
Feature activation changes
Tenant-specific override changes
```

Stale cache after downgrade/revocation is a security/commercial correctness issue.

---

# 84. Request-Level Cache

Within one request:

```text
resolve entitlements once
reuse
```

Do not repeatedly query Subscription/PlanFeature tables for each button-equivalent backend check.

---

# 85. Distributed Cache

Distributed entitlement caching is not required initially.

Start with:

```text
correct database resolution
+
request-level cache
```

Add application/distributed caching only when measurements justify it.

Correct invalidation is more important than cache hit rate.

---

# 86. Entitlement Snapshot in Token

Do not store the full Feature/limit set as long-lived access-token claims.

Reasons:

```text
subscription can change
plan can change
limits can change
downgrade must take effect
```

Frontend and backend resolve dynamic application state separately from authentication.

---

# 87. Plan Claim

A plan key may be present in a response for display, but backend authorization must not treat an old token's plan claim as authoritative.

Authentication and subscription remain separate concerns.

---

# 88. Tenant Switching

When User switches Tenant:

Frontend must refresh:

```text
Subscription
Features
Limits
Permissions
Branches
```

Tenant A entitlements must never leak into Tenant B.

---

# 89. Branch Switching

Subscription is Tenant-level.

Branch switching normally does not change the Tenant Plan.

However Feature usage may be Branch-specific.

Example:

```text
feature exists Tenant-wide
but data is filtered to active Branch
```

Do not duplicate Subscription per Branch unless future product rules require branch-specific subscriptions.

---

# 90. Branch Add-On Future

If future commercial model sells:

```text
extra Branch add-on
```

architecture can extend effective entitlement calculation.

Example:

```text
Base max_branches
+
purchased branch add-ons
```

Do not hardcode add-on logic before product requires it.

---

# 91. Tenant-Specific Overrides

Future support/admin scenarios may need:

```text
temporary Feature enablement
custom enterprise limit
migration exception
```

Potential model:

```text
SubscriptionOverride
```

with:

```text
FeatureKey
Enabled/Limit
EffectiveFrom
EffectiveTo
Reason
```

This is deferred for MVP.

Do not create manual database hacks as a long-term substitute.

---

# 92. Effective Entitlement Precedence

If overrides are later introduced, precedence should be explicitly defined.

Potential future rule:

```text
System safety rule
→ Tenant override
→ PlanVersion entitlement
→ Default deny
```

This is not active until override functionality exists.

---

# 93. Default Deny

Unknown or missing premium Feature:

```text
Denied
```

Do not assume a missing catalog row means:

```text
enabled
```

This mirrors authorization's secure default.

---

# 94. Core Features

Some capabilities are core and need no paid Feature key.

Example:

```text
Login
Basic Tenant workspace
Main Branch
Core appointment flow
```

depending on product scope.

Do not create Feature flags for every line of code.

Feature gating exists for product differentiation, controlled rollout, or entitlement.

---

# 95. Feature Flag vs Entitlement

Subscription Feature and technical Feature Flag are not the same thing.

Entitlement:

```text
Has Tenant purchased/access to capability?
```

Technical feature flag:

```text
Is this code path rolled out/enabled operationally?
```

A future rollout flag system should remain separate.

---

# 96. Kill Switch

Operational emergency disablement may be needed for a Feature.

Example:

```text
public_booking temporarily disabled platform-wide
```

This is a system feature flag/kill switch, not Subscription data.

Effective availability may require:

```text
System Enabled
AND
Tenant Entitled
```

Do not mutate every Plan to perform an emergency shutdown.

---

# 97. Product Catalog vs Runtime Flags

Conceptual separation:

```text
Plan/Feature Catalog
→ commercial entitlement

Operational Feature Flags
→ rollout/safety
```

Operational feature flags are not required by this ADR for MVP.

---

# 98. Feature Audit

Significant entitlement changes should be auditable.

Examples:

```text
Subscription created
Plan changed
Subscription suspended
Subscription reactivated
Manual override added
Manual override removed
```

Audit should include:

```text
TenantId
Actor/System
Old State
New State
Timestamp
External event reference where applicable
```

---

# 99. Subscription History

Do not overwrite all commercial history into one mutable row if that destroys the ability to audit plan changes.

At minimum preserve:

```text
Subscription lifecycle/history
or
audit events
```

Exact history schema may evolve.

---

# 100. Plan Change

Conceptual upgrade:

```text
Current Subscription
↓
New Plan/PlanVersion
↓
Validate commercial transition
↓
Update effective Subscription
↓
Invalidate entitlement cache
↓
Frontend refresh
```

Feature access should change predictably after commit.

---

# 101. Upgrade

Upgrading typically increases entitlement.

Existing data remains valid.

New capabilities become available after:

```text
successful subscription transition
```

according to billing policy.

Do not unlock premium functionality solely because frontend navigated to a success page.

---

# 102. Downgrade

Downgrade flow must:

```text
calculate resulting limits
identify over-limit resources
preserve data
block invalid growth
show remediation
```

Exact timing:

```text
immediate
or
period-end
```

is commercial policy.

---

# 103. Subscription Change Transaction

Subscription state transitions must be transactionally consistent with any internal records required to establish entitlement.

External payment side effects cannot be part of the same database transaction.

Use:

```text
verified provider result
↓
local transaction
↓
commit subscription state
↓
invalidate/cache/event
```

---

# 104. Webhook Idempotency

Billing webhooks may be delivered multiple times.

Local subscription transition must be idempotent using:

```text
provider event ID
or
stable external operation ID
```

when external billing is implemented.

Duplicate webhook must not duplicate Subscription transitions.

---

# 105. Security

A Tenant Owner may manage their Subscription only through allowed product workflows.

They must not be able to:

```text
edit PlanFeature
create arbitrary Feature
set own entitlement
change subscription status directly
```

by manipulating API payloads.

Platform catalog administration is a separate privileged capability.

---

# 106. Platform Catalog Administration

Future platform admin permissions may include:

```text
plans.view
plans.manage
features.view
features.manage
```

These are platform-level, not Tenant Owner permissions.

Tenant Owner may:

```text
view current plan
upgrade/downgrade through approved flow
manage billing information
```

according to product rules.

---

# 107. Testing Strategy

Feature gating requires automated tests at multiple levels.

---

# 108. Entitlement Unit Tests

Test:

```text
Active Subscription + Feature enabled → true
Active Subscription + Feature absent → false
Inactive Subscription → false
Limit configured → correct value
Missing limit → safe default/defined failure
```

---

# 109. Permission + Feature Tests

Matrix:

```text
Permission yes + Feature yes → allowed
Permission no  + Feature yes → denied
Permission yes + Feature no  → denied
Permission no  + Feature no  → denied
```

This verifies the two systems remain independent.

---

# 110. Tenant Isolation Tests

Tenant A's Subscription must never affect Tenant B.

Test:

```text
Tenant A = Salon Pro
Tenant B = Solo

A premium operation in A → allowed
same operation in B → denied
```

even when the same User belongs to both Tenants.

---

# 111. Limit Tests

Example:

```text
max_branches = 1
current branches = 1
create another
→ denied
```

Boundary:

```text
current = 0
limit = 1
→ allowed
```

Concurrency tests are required for critical limits when simultaneous creation could exceed the cap.

---

# 112. Downgrade Tests

Example:

```text
current branches = 4
new limit = 1

read existing → preserved
create new → denied
```

No destructive cleanup occurs automatically.

---

# 113. Cache Invalidation Tests

If entitlement cache is introduced:

```text
Tenant has Feature
↓
Subscription downgraded
↓
cache invalidated
↓
next protected request denied
```

Stale premium access must not persist indefinitely.

---

# 114. Frontend Tests

Angular tests should verify:

```text
Feature enabled → premium UI available
Feature disabled → hidden/locked/upgrade state
Limit reached → create action disabled or explains limit
Tenant switch → entitlement state refreshed
Upgrade completion → state refreshed
```

Backend tests remain authoritative for security.

---

# 115. Integration Tests

Critical APIs should be tested through the real application pipeline.

Examples:

```text
Create Branch
Advanced Report
Public Booking
Premium feature endpoint
```

Test actual entitlement enforcement, not only `IEntitlementService` in isolation.

---

# 116. Registration Test

Registration with Plan selection should verify:

```text
selected valid Plan
↓
Subscription created
↓
Tenant receives expected entitlement
↓
Owner enters workspace
```

Client-supplied Feature tampering must have no effect.

---

# 117. Error UX

Frontend should distinguish:

```text
permission denied
feature unavailable
limit reached
subscription inactive
```

because required user actions differ.

Examples:

```text
permission denied
→ contact admin

feature unavailable
→ upgrade

limit reached
→ upgrade or reduce usage

subscription inactive
→ resolve billing/subscription
```

---

# 118. Observability

Useful entitlement diagnostics:

```text
TenantId
SubscriptionId
PlanKey/Version
FeatureKey
Result
Limit
CurrentUsage where relevant
TraceId
```

Do not log sensitive billing payloads unnecessarily.

---

# 119. Metrics

Potential future metrics:

```text
feature_gate_denied_count
subscription_inactive_denied_count
limit_reached_count
plan_upgrade_count
plan_downgrade_count
```

These may support product analytics and operational diagnostics.

---

# 120. Product Analytics

A gate denial may be a useful product signal:

```text
Tenant attempted Advanced Reports
but Feature unavailable
```

Analytics must remain separate from authorization correctness.

A tracking failure must not change access decisions.

---

# 121. Definition of Done Integration

Any gated Story must explicitly define:

```text
Feature key
Feature kind
Backend enforcement
Frontend behavior
Permission interaction
Limit behavior if applicable
Tenant isolation
Error code
Tests
```

A Story is not Done if:

```text
premium UI is hidden
but API remains callable
```

or:

```text
backend blocks access
but frontend provides no usable upgrade/denial state
```

when UI is in scope.

---

# 122. Rejected Alternatives

Rejected:

```text
Hardcoded plan-name checks
Frontend-only feature gating
Features embedded permanently in auth token
Missing Subscription means allow
Per-Plan database schema
Per-Plan API deployments
Delete data on downgrade
Generic arbitrary JSON entitlement rules
Full metered-billing engine in MVP
Use permissions as subscription features
Use subscription features as permissions
```

---

# 123. Consequences

## Positive

This strategy provides:

```text
Commercial flexibility
Stable application code
Plan-independent business rules
Safe backend enforcement
Future Plan changes
Numeric limits
Tenant-aware capability state
Cleaner Angular UX
Future grandfathering
```

## Negative

It introduces:

```text
Feature catalog
Entitlement resolver
Cache invalidation concerns
Plan/Feature seed management
Limit race conditions
Subscription lifecycle modeling
```

These costs are accepted because subscription is a core SaaS boundary.

---

# 124. Implementation Guidance

Initial backend implementation should focus on:

```text
Plan
Feature
PlanFeature
Subscription
IEntitlementService
Feature constants
Subscription validation
```

Then integrate gates incrementally into real Features.

Do not build a generic pricing engine before product needs it.

---

# 125. Recommended Initial Feature Catalog

Initial catalog may include keys such as:

```text
public_booking
advanced_reports
advanced_notifications
force_appointment

max_branches
max_staff
max_users
```

Only add a Feature when a real product distinction exists.

Do not create placeholder Features for hypothetical future products.

---

# 126. Plan Seed Data

Initial Plans:

```text
solo
solo-pro
salon
salon-pro
```

should be seeded with stable keys.

Commercial labels may remain Persian/localized in presentation data.

Changing display text must not change Plan key.

---

# 127. Seed Idempotency

Plan/Feature seed process must be:

```text
idempotent
safe across deployments
stable by key
```

Do not recreate Plan/Feature rows with new identifiers every startup.

---

# 128. Seed vs Admin UI

MVP may manage Plan/Feature catalog through:

```text
seed/configuration/migrations
```

A full platform Plan administration UI is not required initially.

When commercial operations need runtime editing, add explicit platform administration.

---

# 129. Repository Boundary

`shinera-product` defines:

```text
commercial/product intent
Feature catalog semantics
plan capability decisions
```

`shinera-backend` implements:

```text
entitlement model
enforcement
billing integration
```

`shinera-frontend` implements:

```text
plan display
upgrade UX
feature visibility
limit messaging
```

---

# 130. Relationship to ADR-001

Subscription is Tenant-owned.

Every entitlement query is Tenant-scoped.

Cross-tenant feature leakage is a security defect.

---

# 131. Relationship to ADR-003

Permission and entitlement both participate in protected operations.

Neither replaces the other.

Canonical mental model:

```text
Identity
↓
Tenant Membership
↓
Permission
↓
Entitlement
↓
Business Rule
```

---

# 132. Relationship to ADR-005

Subscription lifecycle timestamps use UTC.

Any date-only commercial rule must explicitly state its business timezone semantics.

Do not use server-local date for expiration.

---

# 133. Example — Create Second Branch

Tenant:

```text
max_branches = 1
```

Current:

```text
1 Branch
```

User:

```text
branches.manage permission
```

Request:

```text
Create Branch
```

Backend:

```text
Authenticate
↓
Tenant Membership
↓
branches.manage
↓
resolve max_branches
↓
current usage = 1
↓
limit reached
↓
reject
```

Error:

```text
subscription.limit_reached
```

---

# 134. Example — Advanced Reports

User:

```text
reports.view
```

Tenant:

```text
advanced_reports = false
```

Result:

```text
deny premium report
```

UI may show:

```text
Upgrade to access Advanced Reports
```

---

# 135. Example — Tenant Switch

User belongs to:

```text
Salon A → Salon Pro
Salon B → Solo
```

Switch:

```text
A → B
```

Frontend must refresh:

```text
Features
Limits
Subscription
Permissions
Branches
```

`advanced_reports` availability may disappear immediately.

No new authentication identity is required solely because Plan changed.

---

# 136. Example — Downgrade

Before:

```text
Plan: Salon Pro
max_branches = 5
Current Branches = 4
```

After:

```text
Plan: Solo
max_branches = 1
```

Result:

```text
4 Branches remain stored
No automatic deletion
New Branch creation denied
UI shows over-limit state
Product workflow guides remediation/upgrade
```

---

# 137. Example — Public Booking

Incoming request:

```text
/book/{tenantSlug}
```

Backend:

```text
Resolve Tenant
↓
Tenant active?
↓
Subscription entitled?
↓
public_booking enabled?
↓
yes → continue
no  → public booking unavailable
```

The public client cannot enable the Feature.

---

# 138. Example — Force Appointment

Future:

```text
Force Appointment
```

may require:

```text
User permission
+
VIP customer rule
+
force_appointment Tenant Feature
+
Appointment concurrency safety
```

These are separate gates.

The Feature does not replace VIP eligibility or permission.

---

# 139. Revisit Conditions

Revisit this ADR when:

```text
usage-based billing becomes real
add-ons become commercial products
enterprise custom contracts require overrides
plan grandfathering is required in production
seat-based billing is introduced
branch-specific subscriptions are introduced
billing provider semantics become complex
feature rollout flags are added
```

Until then, keep entitlement logic simple and explicit.

---

# 140. Final Decision Summary

Shinera subscription architecture is:

```text
Tenant
↓
Subscription
↓
Plan / PlanVersion
↓
PlanFeature
↓
Feature
↓
Effective Entitlements
```

Application code uses:

```text
Feature capability
or
Feature limit
```

and does not depend on commercial Plan names.

Core rules:

```text
Backend is authoritative
Frontend gating is UX only
Default deny for unavailable Features
Permission and Entitlement are separate
Subscription belongs to Tenant
Limits are first-class
Downgrade preserves data
Over-limit blocks new growth
Dynamic entitlement is not stored as permanent auth-token claims
```

This architecture allows Shinera to change pricing and packaging without rewriting core business logic.

Commercial Plans may evolve.

Feature keys remain the stable contract between product packaging and application behavior.
