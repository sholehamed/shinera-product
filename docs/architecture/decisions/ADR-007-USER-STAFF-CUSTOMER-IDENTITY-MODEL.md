# ADR-007 — User, Staff and Customer Identity Model

**Project:** Shinera  
**Status:** Accepted  
**Date:** 2026-10-05  
**Decision Type:** Architecture / Identity / Domain Modeling  
**Scope:** Authentication, Staff, Customers, Tenancy, Authorization, Public Booking  
**Related Documents:**
- `SHINERA-PRD.md`
- `SHINERA-PRODUCT-BACKLOG.md`
- `PROJECT-INSTRUCTIONS.md`
- `ARCHITECTURE-OVERVIEW.md`
- `DEFINITION-OF-DONE.md`
- `ADR-001-MULTI-TENANCY-STRATEGY.md`
- `ADR-002-AUTHENTICATION-WITH-OPENIDDICT.md`
- `ADR-003-AUTHORIZATION-AND-PERMISSION-MODEL.md`
- `ADR-006-SUBSCRIPTION-AND-FEATURE-GATING.md`

---

# 1. Context

Shinera has three concepts that may refer to the same real-world person but represent different responsibilities:

```text
User
Staff
Customer
```

Examples:

```text
Sara
→ has a Shinera login
→ works as a hairstylist in Salon A
→ may also book a service as a Customer

Neda
→ has no Shinera account
→ is a Customer of Salon A

Ali
→ is a Staff member
→ salon owner created his Staff profile
→ Ali has not created a login account yet
```

If these concepts are merged into one entity, several problems appear:

- Customers would inherit internal workspace concepts.
- Staff without login accounts would be difficult to represent.
- The same person being a Customer of multiple salons would become awkward.
- Authentication concerns would leak into CRM data.
- Tenant-specific notes/history could become global accidentally.
- Internal permissions could become confused with Customer access.

Therefore Shinera separates:

```text
Authentication Identity
Operational Staff Identity
CRM Customer Identity
```

---

# 2. Decision

Shinera will model:

```text
User
Staff
Customer
```

as separate entities.

Relationships:

```text
User
 ├── may link to Staff
 └── may link to Customer

Staff
 └── UserId?   optional

Customer
 └── UserId?   optional
```

The same `User` may be linked to multiple Tenant-specific Staff and/or Customer records where business rules permit.

---

# 3. Core Mental Model

Use this rule:

```text
User
= Who can authenticate?

Staff
= Who provides/participates in salon operations?

Customer
= Who receives services and owns customer history?
```

These are separate questions.

---

# 4. User

`User` is a global Shinera authentication identity.

Conceptually:

```text
User
├── Id
├── FirstName
├── LastName
├── Phone
├── Email
├── PasswordHash
├── IsActive
├── LastLoginAt
└── ...
```

User belongs to the authentication/security domain.

User is not automatically:

```text
Staff
Customer
Tenant Owner
Tenant Member
```

Those relationships are explicit.

---

# 5. User Scope

`User` is platform-level identity data.

A User may participate in multiple Tenants.

Example:

```text
User U1
├── Tenant A → Owner
├── Tenant B → Staff
└── Tenant C → Customer
```

This is valid.

Tenant access is represented by domain-specific relationships, not by duplicating authentication Users per Tenant.

---

# 6. Staff

`Staff` represents a service provider or operational employee inside a Tenant.

Conceptually:

```text
Staff
├── Id
├── TenantId
├── UserId?
├── DisplayName
├── Phone
├── IsActive
├── ...
├── StaffBranch
├── StaffService
└── StaffSchedule
```

Staff is Tenant-owned.

---

# 7. Staff May Exist Without User

A salon must be able to create Staff before that Staff has a Shinera login.

Example:

```text
Owner creates:
Staff = "Sara"
```

Sara can immediately:

```text
be assigned to Branch
provide Services
receive Appointments
have a Schedule
```

even if:

```text
UserId = null
```

This is required for practical salon onboarding.

---

# 8. Staff Login Is Optional

When a Staff member needs access to Shinera:

```text
Staff
↓
linked User
↓
TenantMembership
↓
Role / Permissions
```

The Staff record does not itself authenticate.

Authentication always belongs to User.

---

# 9. Staff.UserId

`Staff.UserId` is optional.

Conceptually:

```text
Staff.UserId = null
```

means:

```text
Operational Staff profile exists
but no login identity is linked.
```

When linked:

```text
Staff.UserId = User.Id
```

the authorization system can evaluate `Own` scope.

---

# 10. Own Scope for Staff

ADR-003 defines `Own` scope.

For Staff resources, Own may use:

```text
Staff.UserId == CurrentUser.Id
```

Examples:

```text
Staff sees own Appointments
Staff sees own Schedule
Staff updates own profile
```

This only works when Staff is linked to User.

---

# 11. Staff Is Not a Role

Do not confuse:

```text
Staff entity
```

with:

```text
Staff role
```

They represent different concepts.

`Staff` entity:

```text
operational service provider
```

`Staff` Role:

```text
permission bundle
```

A User could theoretically have a Staff record while holding a different Role configuration.

---

# 12. TenantMembership and Staff

If a Staff-linked User may enter the internal workspace, they require:

```text
TenantMembership
```

in addition to:

```text
Staff.UserId
```

The link:

```text
Staff → User
```

does not automatically grant workspace access.

Authorization remains explicit.

---

# 13. BranchMembership and Staff

Staff operational assignment:

```text
StaffBranch
```

answers:

```text
At which Branch can this Staff provide services?
```

User Branch access:

```text
BranchMembership
```

answers:

```text
Which Branch workspace may this User operate in?
```

These may correlate but are not the same concept.

---

# 14. Why StaffBranch and BranchMembership Stay Separate

Example:

```text
Sara provides services in Branch A and Branch B
```

but her workspace account may initially only allow:

```text
Branch A
```

or vice versa for administrative reasons.

Therefore:

```text
Staff operational assignment
!=
User authorization assignment
```

---

# 15. Customer

`Customer` represents a person receiving services from a specific Tenant.

Conceptually:

```text
Customer
├── Id
├── TenantId
├── UserId?
├── FirstName
├── LastName
├── Mobile
├── Email?
├── Notes
├── IsActive
└── ...
```

Customer is Tenant-owned CRM data.

---

# 16. Customer Is Not User

Most salon Customers may initially have no Shinera account.

Example:

```text
Receptionist creates Customer:
Neda
0912...
```

Neda may:

```text
receive Appointments
have Payment history
have Notes
appear in CRM
```

without ever creating a login.

Therefore Customer cannot be modeled as User only.

---

# 17. Customer.UserId

`Customer.UserId` is optional.

```text
Customer.UserId = null
```

means:

```text
CRM Customer exists
but no authenticated Shinera account is linked.
```

Later:

```text
Customer.UserId = User.Id
```

may be established after verified account linking.

---

# 18. Customer Can Be Tenant-Specific

A real-world person may be Customer of multiple salons.

Example:

```text
User U1
├── Customer C1 in Tenant A
├── Customer C2 in Tenant B
└── Customer C3 in Tenant C
```

Each Customer record has its own:

```text
Appointment history
Notes
VIP state
Loyalty state
Payments
Tenant-specific metadata
```

This data must not be shared automatically across Tenants.

---

# 19. Why Customer Is Tenant-Owned

Salon A may record:

```text
Hair color preference
Notes
VIP status
No-show history
```

Salon B must not automatically see that data.

Therefore:

```text
Customer
```

is a Tenant-owned entity even if linked to the same global User.

This protects Tenant isolation.

---

# 20. Customer Authentication

If Customers are allowed to log in:

```text
Authentication
→ User
```

not:

```text
Customer password
```

Customer access is then resolved through:

```text
User
→ Customer record(s)
```

for the relevant Tenant.

---

# 21. Customer Does Not Need TenantMembership for CRM Existence

A Customer record does not require:

```text
TenantMembership
```

simply to exist.

Customer is not an internal workspace member.

This avoids treating every salon customer as an employee/member of the Tenant.

---

# 22. Customer Workspace Access

Normal Customer users do not enter the internal Owner/Staff workspace.

They use customer-facing surfaces such as:

```text
Public Booking
Customer Portal
My Appointments
Profile
```

if/when those features are implemented.

Internal `TenantMembership` remains intended for business/workspace participation.

---

# 23. Customer Authorization Context

Customer-facing authorization is primarily:

```text
Own
```

Examples:

```text
view own Appointments
cancel own Appointment when allowed
view own Profile
update own contact information
```

This authorization derives from:

```text
Current User
↓
linked Customer
↓
Appointment.CustomerId
```

not internal Staff roles.

---

# 24. Internal vs Customer Authorization

Keep two mental domains:

```text
Internal Workspace Authorization
→ Owner / Admin / Receptionist / Staff

Customer Authorization
→ Customer acting on own customer data
```

The same authorization engine may support both, but permission catalogs should not blur their meaning.

---

# 25. Potential Customer Permissions

Future customer-facing keys may include:

```text
customer.profile.view_own
customer.profile.update_own
customer.appointments.view_own
customer.appointments.create
customer.appointments.cancel_own
```

Exact customer permission design can be introduced when customer login becomes active.

Do not preload unnecessary IAM complexity in MVP.

---

# 26. Public Booking Without User

Public booking must support Customers without Shinera accounts.

Conceptual:

```text
Public booking
↓
identify customer
↓
create or resolve Customer
↓
create Appointment
```

No User account is required.

---

# 27. Public Booking With Existing User

If the public visitor is authenticated:

```text
User
↓
find linked Customer in target Tenant
```

If none exists:

```text
create Customer
or
link verified existing Customer
```

according to safe identity rules.

---

# 28. Customer Matching

Initial Customer matching may use:

```text
TenantId + normalized Mobile
```

where product rules allow.

Potential uniqueness:

```text
UNIQUE(TenantId, NormalizedMobile)
```

if the business rule is one Customer record per phone per Tenant.

This does not imply global uniqueness across Shinera.

---

# 29. Global Phone Uniqueness

Authentication User phone may be globally unique if used as the login identifier.

Customer phone does not need global uniqueness.

Therefore:

```text
User.Phone
```

and:

```text
Customer.Mobile
```

have different uniqueness semantics.

---

# 30. Customer Linking Security

Never link:

```text
Customer
→ User
```

only because the entered phone/email strings match.

Linking requires verification.

Examples:

```text
authenticated User already verified that phone
OTP verification
explicit secure claim process
```

Avoid account takeover through guessed contact information.

---

# 31. Existing Customer Claim Flow

Possible future flow:

```text
User logs in/registers
↓
enters verified mobile
↓
system finds unlinked Customer in Tenant
↓
verification/business rules pass
↓
Customer.UserId = User.Id
```

If multiple records are ambiguous:

```text
do not auto-link silently
```

Use a controlled merge/claim process.

---

# 32. Customer Merge

Duplicate Customer records may occur.

A future merge operation may consolidate:

```text
Appointments
Payments
Notes
Customer metadata
```

Customer merge is a CRM function.

It is separate from:

```text
User account merge
```

Do not treat linking to the same User as automatic Customer merge.

---

# 33. User Account Merge

Merging authentication User accounts is a security-sensitive platform operation and is outside MVP.

If two Users claim the same identity unexpectedly:

```text
do not merge automatically
```

---

# 34. Person Can Be Both Staff and Customer

This is explicitly allowed.

Example:

```text
User U1
├── Staff S1 in Tenant A
└── Customer C1 in Tenant A
```

The same person may:

```text
work at salon
and
receive salon services
```

These records remain separate.

---

# 35. Why Staff and Customer Are Not One "Person" Record

A generic Person model was considered.

Example:

```text
Person
├── User?
├── StaffProfile?
└── CustomerProfile?
```

This may look normalized but introduces complexity:

- Tenant ownership becomes less obvious.
- Staff and Customer lifecycle differ.
- Notes/privacy differ.
- Customer may exist in many Tenants.
- Staff operational fields differ from CRM fields.
- authorization becomes harder to reason about.

For MVP:

```text
separate Staff and Customer entities
```

is clearer.

---

# 36. Shared Contact Data Duplication

Because Staff and Customer are separate, some fields may be duplicated:

```text
Name
Phone
Email
Image
```

This is accepted.

These fields represent domain snapshots/profile data with different ownership.

Do not over-normalize identity data into a universal Person table prematurely.

---

# 37. User Profile vs Staff Profile

User profile answers:

```text
Who is the account holder?
```

Staff profile answers:

```text
How is this provider represented operationally in this Tenant?
```

Example:

User:

```text
Legal/preferred account name
Global avatar
Login phone
```

Staff:

```text
Salon display name
Professional title
Bio
Service assignments
Branch assignments
Schedule
```

---

# 38. User Profile vs Customer Profile

User:

```text
global account identity
```

Customer:

```text
Tenant-specific CRM record
```

Customer may include:

```text
preferred name
salon notes
VIP state
loyalty status
service preferences
```

These should not become global User profile data.

---

# 39. Data Privacy

Customer notes belong to the Tenant.

They are not automatically visible to:

```text
the Customer User
other Tenants
other unrelated Staff
```

Visibility follows explicit permission/business policy.

Do not expose internal CRM notes simply because Customer.UserId matches current User.

---

# 40. Staff Internal Data

Similarly, Staff internal HR/operational notes should not automatically become User-visible profile data.

Entity linking does not imply all fields are shared.

---

# 41. User Deactivation

If:

```text
User.IsActive = false
```

the User cannot authenticate/continue sessions according to ADR-002.

Linked Staff and Customer records do not need to be deleted.

Example:

```text
Staff remains in historical Appointment records.
Customer history remains intact.
```

---

# 42. Staff Deactivation

If:

```text
Staff.IsActive = false
```

the Staff cannot receive new normal Appointments.

This does not necessarily disable:

```text
linked User login
```

because User may hold another role or participate elsewhere.

User and Staff lifecycle are independent.

---

# 43. Customer Deactivation

If:

```text
Customer.IsActive = false
```

the CRM Customer may be blocked from new business actions according to product rules.

This does not necessarily deactivate the global User account.

---

# 44. TenantMembership Revocation

Revoking:

```text
TenantMembership
```

removes internal workspace access.

It does not automatically delete:

```text
Staff
Customer
```

records.

If Staff login access is removed but Staff remains employed operationally:

```text
Staff.UserId may remain linked
```

while no valid TenantMembership grants workspace access.

---

# 45. Staff Employment Lifecycle

Staff lifecycle may include:

```text
Active
Inactive
Archived
```

or equivalent.

Historical Appointments must continue referencing the Staff record.

Hard delete should generally be avoided when history exists.

---

# 46. Customer Lifecycle

Customer may be:

```text
Active
Archived
Blocked
```

depending on future product rules.

Historical service/payment data must be preserved.

---

# 47. Ownership Relationships

Conceptually:

```text
User
→ platform/global

Staff
→ Tenant-owned

Customer
→ Tenant-owned

TenantMembership
→ User ↔ Tenant

BranchMembership
→ User ↔ Branch

StaffBranch
→ Staff ↔ Branch

StaffService
→ Staff ↔ Service
```

Each relationship answers a distinct business question.

---

# 48. Internal User Onboarding

When Owner invites a Staff member to login:

```text
Existing Staff
↓
invite / identify User
↓
link Staff.UserId
↓
create TenantMembership
↓
assign Role
↓
assign BranchMembership where required
```

Linking alone is not enough.

---

# 49. Invite Before User Exists

A Staff invitation may exist before the User account is created.

Potential flow:

```text
Owner invites mobile/email
↓
Invitation record
↓
recipient registers/authenticates
↓
verify invitation
↓
link/create User
↓
link Staff
↓
create Membership
```

Invitation architecture can be designed when implemented.

---

# 50. Owner Is Also User

Tenant Owner is always:

```text
User
+
TenantMembership
+
Owner role/permissions
```

Owner does not need to be Staff unless they actually provide services.

If Owner is also a stylist:

```text
User
+
TenantMembership
+
Owner Role
+
Staff linked to same User
```

This is valid.

---

# 51. Receptionist

Receptionist is usually:

```text
User
+
TenantMembership
+
Receptionist Role
```

A Receptionist does not require a Staff record unless they also provide services.

---

# 52. Admin

Admin is:

```text
User
+
TenantMembership
+
Admin Role
```

Again, no Staff record is required merely because the person works for the salon.

---

# 53. Staff Login Example

Sara:

```text
User U100
TenantMembership TM1
Role = Staff

Staff S55
UserId = U100
TenantId = T1
```

Authorization:

```text
appointments.view + Own
```

can resolve:

```text
Appointment.StaffId == S55
```

through the User linkage.

---

# 54. Staff Without Login Example

Maryam:

```text
Staff S56
UserId = null
TenantId = T1
```

She may:

```text
have Schedule
provide Services
receive Appointments
```

but cannot:

```text
authenticate
open Staff workspace
```

until linked to a User and Membership.

---

# 55. Customer Without Login Example

Neda:

```text
Customer C300
UserId = null
TenantId = T1
```

She may:

```text
have Appointment history
Payment history
Notes
VIP status
```

with no Shinera account.

---

# 56. Customer With Login Example

Neda later registers:

```text
User U900
```

After verified linking:

```text
Customer C300.UserId = U900
```

Now customer-facing app may resolve:

```text
My Appointments
→ Customer C300
```

inside Tenant T1.

---

# 57. Same User Across Multiple Customer Records

Neda may visit several salons:

```text
User U900

Tenant T1:
Customer C300 → U900

Tenant T2:
Customer C811 → U900

Tenant T3:
Customer C922 → U900
```

Each Customer record remains Tenant-isolated.

---

# 58. Same User Across Multiple Staff Records

A specialist may work in multiple independent businesses.

Example:

```text
User U100

Tenant T1:
Staff S10 → U100

Tenant T2:
Staff S77 → U100
```

This is allowed.

Each Staff record has:

```text
different Schedule
Services
Branches
Profile
```

---

# 59. Staff in Multiple Branches of Same Tenant

Within one Tenant:

```text
one Staff record
+
multiple StaffBranch links
```

is preferred over:

```text
one Staff record per Branch
```

unless future product requirements require branch-specific staff profiles.

This keeps Staff identity Tenant-wide.

---

# 60. Customer Across Branches

Customer is Tenant-wide by default.

A Customer visiting:

```text
Branch A
Branch B
```

within the same Tenant should normally remain:

```text
one Customer record
```

Appointments carry Branch ownership.

Do not duplicate Customer per Branch by default.

---

# 61. Why Customer Is Tenant-Level, Not Branch-Level

CRM relationship belongs to the business.

Benefits:

- unified history,
- duplicate reduction,
- cross-branch customer recognition,
- cleaner loyalty/VIP behavior.

Branch-specific access still applies through authorization.

---

# 62. Customer Notes and Branch Scope

Even though Customer is Tenant-wide, a Branch-scoped User may have limited visibility.

Authorization may decide:

```text
can view Customer because they have an Appointment in accessible Branch
```

or other product rules.

Tenant ownership and authorization visibility are separate concepts.

---

# 63. Customer Search

Customer search should remain Tenant-scoped.

Conceptually:

```text
TenantId
+
NormalizedMobile / Name
```

Do not expose global User/customer search to salon users.

---

# 64. Appointment Relationships

Appointment references:

```text
Tenant
Branch
Customer
Staff
Service
```

Customer and Staff must belong to the same Tenant as Appointment.

If Staff/User or Customer/User links exist, those are not the Appointment ownership keys.

---

# 65. Appointment Own Scope — Staff

For internal Staff:

```text
Own Appointment
```

means appointment assigned to linked Staff.

Conceptually:

```text
Appointment.Staff.UserId == CurrentUser.Id
```

or an equivalent StaffId resolution.

---

# 66. Appointment Own Scope — Customer

For customer-facing user:

```text
Own Appointment
```

means:

```text
Appointment.Customer.UserId == CurrentUser.Id
```

within the relevant Tenant.

These two notions of Own are contextual and distinct.

---

# 67. Avoid Ambiguous Generic Own Logic

Do not implement:

```text
Appointment.UserId == CurrentUser.Id
```

because Appointment has multiple person roles.

Own must specify:

```text
Staff ownership
or
Customer ownership
```

depending on authorization context.

---

# 68. Customer Login Routing

Authenticated Customer should enter a customer-facing experience.

Internal workspace routing should require:

```text
TenantMembership
```

and appropriate internal permissions.

A Customer User without TenantMembership must not accidentally enter salon administration.

---

# 69. User Can Have Both Experiences

If a User is:

```text
Owner in Tenant A
Customer in Tenant B
```

the product may offer:

```text
Business Workspace
Customer Area
```

based on available relationships.

Authentication identity remains the same.

---

# 70. Workspace Selector

Internal workspace selector uses:

```text
TenantMembership
```

not Customer records.

Customer-facing salon selection may use:

```text
Customer links / public business context
```

These are separate navigation models.

---

# 71. Customer Tenant Context

Customer-facing operations still need a Tenant context.

Example:

```text
My Appointment in Salon A
```

must resolve:

```text
Tenant A
+
Customer record linked to Current User
```

The User cannot choose arbitrary CustomerId.

---

# 72. Customer Claim Security

For:

```text
GET /customer/me
```

backend resolves Customer from:

```text
Current User
+
Current public/customer Tenant
```

not:

```text
customerId supplied by browser
```

This mirrors ADR-001 tenant safety.

---

# 73. Customer Mobile Change

Changing:

```text
Customer.Mobile
```

does not automatically change:

```text
User.Phone
```

and vice versa.

If a linked User wants to synchronize verified contact data, use an explicit workflow.

Do not create hidden cross-entity updates.

---

# 74. Staff Mobile Change

The same applies to Staff.

Staff business contact and User login contact may differ.

Avoid automatic synchronization unless explicitly defined.

---

# 75. Profile Images

Potentially:

```text
User Avatar
Staff Profile Image
Customer Image
```

may be separate.

The same underlying file may be reused intentionally, but domain semantics remain distinct.

---

# 76. Deletion

Deleting User must not cascade destructively into historical Staff/Customer data.

Prefer:

```text
deactivate/anonymize User
```

with carefully controlled unlinking where required.

Historical business records must remain valid.

---

# 77. User Anonymization

Future privacy/legal workflows may anonymize User identity while preserving business records where legally permitted.

Staff/Customer data retention has separate business/legal policies.

Do not design hard cascade deletion between these entities.

---

# 78. Foreign Keys

Conceptually:

```text
Staff.UserId → User.Id nullable
Customer.UserId → User.Id nullable
```

Delete behavior should not cascade business records.

Preferred:

```text
Restrict
or
SetNull
```

depending on final privacy/deletion workflow.

Avoid:

```text
Cascade delete Staff/Customer
```

from User deletion.

---

# 79. Unique Staff-User Link

Within one Tenant, a User should normally link to at most one active Staff record.

Potential constraint:

```text
UNIQUE(TenantId, UserId)
WHERE UserId IS NOT NULL
```

where provider support allows.

This prevents duplicate operational Staff identities for one account in the same Tenant.

---

# 80. Unique Customer-User Link

Within one Tenant, a User should normally link to at most one active Customer identity.

Potential constraint:

```text
UNIQUE(TenantId, UserId)
WHERE UserId IS NOT NULL
```

This avoids ambiguous customer-facing identity.

Customer merge handles duplicates rather than keeping multiple linked active records.

---

# 81. Same User as Staff and Customer

The constraints are per entity type.

This remains valid:

```text
Staff(T1, U1)
Customer(T1, U1)
```

No conflict exists.

---

# 82. Linking Validation

When linking Staff to User:

Validate:

```text
User exists
User active enough for linking
no conflicting Staff link in Tenant
business invitation/verification valid
```

When linking Customer to User:

Validate:

```text
User exists
contact/account ownership verified
no conflicting active Customer link in Tenant
```

---

# 83. Staff User Link Permission

Only authorized internal users/processes may link Staff accounts.

Potential permission:

```text
staff.invite
staff.link_user
```

Do not let ordinary Staff arbitrarily claim another Staff record.

---

# 84. Customer Self-Link

Customer linking may be self-service only after strong verification.

This is a customer identity flow, not internal Staff permission.

---

# 85. Customer VIP

VIP is a Customer attribute/business concept.

Example:

```text
Customer.IsVip
or
CustomerTier
```

It does not belong on User.

Reason:

A person may be VIP in Salon A and ordinary in Salon B.

---

# 86. Loyalty

Future loyalty state belongs to:

```text
Customer within Tenant
```

not global User.

This preserves business ownership.

---

# 87. Customer Preferences

Tenant-specific preferences such as:

```text
preferred Staff
favorite Services
allergies/notes where legally appropriate
```

belong to Customer domain.

Global account preferences such as:

```text
language
theme
notification account preferences
```

belong to User.

---

# 88. Notifications

Notification recipient identity may come from different sources.

Internal:

```text
User contact
```

Customer appointment reminder:

```text
Customer contact
```

If linked, the system may choose verified User contact according to notification policy.

Do not assume all Customer notifications require User account.

---

# 89. Audit Actor

Audit actor is normally:

```text
UserId
```

because User is the authenticated identity.

Business target may be:

```text
StaffId
CustomerId
```

Example:

```text
User U1 updates Staff S5 schedule
```

Audit records:

```text
ActorUserId = U1
Target = Staff S5
```

---

# 90. Public Anonymous Actor

Public booking without login has no User actor.

Audit may record:

```text
ActorType = Public/Anonymous
CustomerId
TenantId
TraceId
```

according to audit design.

Do not create fake User accounts for anonymous public bookings.

---

# 91. System Actor

Background/system operations may use:

```text
ActorType = System
```

rather than a fake User.

User remains human/service authentication identity.

---

# 92. Authentication Claims

Tokens identify:

```text
User
```

not Staff or Customer as the primary subject.

Application resolves domain links after authentication.

This keeps authentication stable if Staff/Customer relationships change.

---

# 93. Why StaffId Is Not Primary `sub`

OpenID Connect `sub` should represent the User identity.

A Staff record can:

```text
change
be deactivated
exist in multiple Tenants
```

while User identity remains stable.

Therefore StaffId should not replace UserId as authentication subject.

---

# 94. Why CustomerId Is Not Primary `sub`

Same reason.

A User may have multiple Customer records across Tenants.

Token subject remains User.

---

# 95. `/auth/me`

Current-user bootstrap may include:

```text
User profile
TenantMemberships
available Staff links
customer relationships when needed
```

but should not return every Customer/Staff record globally without purpose.

Keep payload aligned with active UX.

---

# 96. Internal Workspace Bootstrap

For active internal Tenant:

```text
User
TenantMembership
Role/Permissions
BranchMemberships
Staff link?
Features
```

may be resolved.

If User has linked Staff:

```text
CurrentStaffId
```

may be included for Own-scope UX.

---

# 97. Customer Bootstrap

Customer-facing bootstrap may resolve:

```text
User
Current Tenant
CustomerId
Customer profile
own Appointment summary
```

No internal Role/permission graph is required unless customer IAM uses shared permission infrastructure.

---

# 98. Entity IDs

Use separate identifiers:

```text
UserId
StaffId
CustomerId
```

Do not reuse one Guid as all three concepts.

Even if a person maps one-to-one today, the identities have different lifecycles and cardinalities.

---

# 99. Naming in Code

Avoid ambiguous properties:

```text
PersonId
AccountId
MemberId
```

when the domain meaning is actually:

```text
UserId
StaffId
CustomerId
TenantMembershipId
```

Explicit naming prevents authorization bugs.

---

# 100. DTO Design

Do not expose UserId unnecessarily in operational APIs.

Example Staff DTO may include:

```text
hasAccount
```

rather than exposing global User identity unless admin workflow requires it.

Customer DTO similarly should not leak authentication details unnecessarily.

---

# 101. Privacy Between Tenants

A Tenant must not learn:

```text
which other salons the same User visits
where Staff also works
whether Customer has global account relationships elsewhere
```

unless a future explicit platform feature permits it.

Global User linkage is internal platform identity information.

---

# 102. Global Customer Search Is Rejected

Salon users cannot search:

```text
all Shinera Users
all Shinera Customers
```

to find people across businesses.

Customer search remains Tenant-scoped.

---

# 103. Global Staff Search Is Rejected by Default

Similarly, a salon cannot discover a User's Staff records in unrelated Tenants.

Cross-business professional marketplace features would require explicit future architecture/product rules.

---

# 104. Customer Data Import

Imported CRM Customers are created as:

```text
Customer
UserId = null
```

They do not require User account creation.

This supports migration from spreadsheets/legacy systems.

---

# 105. Staff Data Import

Imported Staff are:

```text
Staff
UserId = null
```

until invited/linked.

Do not generate credentials during bulk import automatically.

---

# 106. Registration Owner Creation

Owner registration creates:

```text
User
Tenant
TenantMembership
Owner Role
```

It creates Staff only if onboarding/business setup says the Owner personally provides services.

Do not assume every Owner is a Staff member.

---

# 107. Customer Registration

Customer self-registration creates:

```text
User
```

and then may:

```text
link/create Customer per Tenant as needed
```

It does not create TenantMembership.

---

# 108. One Global User, Many Contexts

Canonical model:

```text
                User U1
             /     |      \
            /      |       \
           ▼       ▼        ▼
 Tenant A Staff  Tenant B Customer  Tenant C Membership
```

This supports one login identity across Shinera without sharing Tenant-owned business data.

---

# 109. Domain Boundaries

Authentication domain owns:

```text
User
credentials
sessions
```

Tenancy/IAM owns:

```text
TenantMembership
BranchMembership
Roles
Permissions
```

Workforce domain owns:

```text
Staff
StaffBranch
StaffService
Schedule
```

CRM domain owns:

```text
Customer
CustomerNotes
Customer history
VIP/Loyalty
```

Appointments connect:

```text
Staff
Customer
Service
Branch
```

---

# 110. Authorization Matrix

Conceptually:

| Actor | User | TenantMembership | Staff | Customer |
|---|---:|---:|---:|---:|
| Owner | Yes | Yes | Optional | Optional |
| Admin | Yes | Yes | Optional | Optional |
| Receptionist | Yes | Yes | Usually No | Optional |
| Logged-in Staff | Yes | Yes | Yes | Optional |
| Staff without login | No | No | Yes | Optional |
| Logged-in Customer | Yes | No | No | Yes |
| Anonymous Customer | No | No | No | Yes |
| Owner who is stylist | Yes | Yes | Yes | Optional |
| Staff who books service | Yes | Yes | Yes | Yes |

This table describes identity relationships, not every possible product rule.

---

# 111. Customer and TenantMembership

Default decision:

```text
Customer relationship
!=
TenantMembership
```

This is important.

TenantMembership is reserved for internal business workspace participation.

If a future product requires customer membership-like features, create an explicit customer access model rather than overloading internal membership semantics.

---

# 112. Why This Matters

If every Customer became TenantMembership:

- membership tables could contain huge customer populations,
- internal workspace selection could show salons visited as "workspaces",
- role semantics become confusing,
- staff/admin IAM mixes with CRM users,
- permission queries become noisier.

Therefore the concepts stay separate.

---

# 113. Customer Portal Context

Customer-facing URL/context may be:

```text
Salon public domain/slug
+
authenticated User
```

Backend resolves:

```text
Tenant
+
Customer linked to User
```

This does not require internal workspace membership.

---

# 114. Future Marketplace

If Shinera later becomes a cross-salon marketplace:

```text
User
```

is the natural global consumer identity.

Tenant-specific Customer records remain the salons' CRM projection of that User.

This architecture supports a marketplace without exposing CRM data across Tenants.

---

# 115. Future Professional Profile

If Shinera later creates a global professional marketplace profile, it should be a new explicit concept such as:

```text
ProfessionalProfile
```

rather than turning Staff into a global entity.

Current Staff remains Tenant-owned.

---

# 116. Hard Delete

Avoid hard deletion of Staff/Customer with historical Appointments/Payments.

Prefer lifecycle state.

User deletion/anonymization must not cascade historical business records.

---

# 117. Soft Delete

If Staff/Customer use soft delete:

```text
Tenant filters
+
soft-delete filters
```

must remain independent.

Historical Appointment navigation may intentionally include archived identity snapshots/records.

---

# 118. Historical Names

Appointment/history screens may need stable historical display even if Staff/Customer names later change.

Potential future solution:

```text
snapshot display name on Appointment
```

or historical audit/versioning.

This is not required by this ADR but should be considered where legal/reporting history needs it.

---

# 119. Identity Snapshot Tradeoff

Current primary relationships remain FK-based.

Do not duplicate every identity field into Appointment prematurely.

Add snapshots only for real historical/reporting requirements.

---

# 120. Customer Notes Security

Customer internal Notes require explicit internal permissions.

A logged-in Customer must not automatically receive:

```text
Customer.Notes
```

because they are linked to that record.

Public/customer DTOs and internal CRM DTOs must be separate where fields differ.

---

# 121. Staff Internal Fields Security

Likewise, Staff public profile DTO should not expose:

```text
internal permissions
private employment notes
authorization metadata
```

just because Staff is linked to User.

---

# 122. API Separation

Potential APIs:

Internal:

```text
/api/customers
/api/staff
```

Customer-facing:

```text
/api/customer/me
/api/customer/appointments
```

Public:

```text
/api/public/{tenantSlug}/...
```

Exact routes may differ.

The separation reflects authorization context.

---

# 123. Customer Creation Source

Customer may originate from:

```text
Receptionist manual creation
Appointment booking
Public booking
Data import
Customer self-registration
```

All sources should converge on the same Tenant-owned Customer model.

---

# 124. Customer Source Metadata

A future Customer may track:

```text
Source
CreatedBy
AcquisitionChannel
```

These fields belong to CRM/analytics.

They do not affect authentication identity.

---

# 125. Staff Creation Source

Staff may originate from:

```text
Owner creation
Import
Invitation
Owner self-setup
```

The Staff model remains the same.

---

# 126. Account Link Status

UI may expose:

```text
Account linked
Invitation pending
No account
```

for Staff.

This is derived from linking/invitation state.

Do not treat `UserId != null` as the only future invitation status if invitations are implemented.

---

# 127. Customer Account Status

Similarly customer UI may show:

```text
Registered Customer
Guest Customer
```

without changing CRM identity.

---

# 128. Testing Strategy

This identity model requires tests for:

```text
Staff without User
Staff linked to User
Customer without User
Customer linked to User
same User across multiple Tenants
same User as Staff and Customer
Tenant isolation
Own scope
linking verification
membership separation
```

---

# 129. Staff Own-Scope Test

Setup:

```text
User U1
Staff S1 → U1

User U2
Staff S2 → U2
```

Permission:

```text
appointments.view + Own
```

Expected:

```text
U1 sees S1 Appointments
U1 does not see S2 Appointments
```

---

# 130. Customer Own-Scope Test

Setup:

```text
User U1
Customer C1 → U1

User U2
Customer C2 → U2
```

Expected customer-facing behavior:

```text
U1 sees C1 Appointments
U1 cannot access C2 Appointments
```

---

# 131. Cross-Tenant Customer Test

Setup:

```text
User U1

Tenant A:
Customer CA → U1

Tenant B:
Customer CB → U1
```

Accessing Tenant A customer context must not reveal:

```text
Tenant B notes
Appointments
VIP state
Payments
```

---

# 132. Cross-Tenant Staff Test

Setup:

```text
User U1

Tenant A:
Staff SA → U1

Tenant B:
Staff SB → U1
```

Tenant A Own scope resolves:

```text
SA
```

not:

```text
SB
```

---

# 133. Staff Link Test

Attempting to link:

```text
same User
to second active Staff
inside same Tenant
```

should be rejected unless a future explicit business reason exists.

---

# 134. Customer Link Test

Attempting to link:

```text
same User
to second active Customer
inside same Tenant
```

should be rejected/require merge resolution.

---

# 135. Customer Without Account Test

Customer CRUD and Appointment creation must work when:

```text
Customer.UserId = null
```

This is a core requirement.

---

# 136. Staff Without Account Test

Staff schedule and booking must work when:

```text
Staff.UserId = null
```

This is also core.

---

# 137. Owner-Staff Test

Owner User may be linked to Staff.

Expected:

```text
Owner permissions remain Owner permissions
Own Staff identity may also resolve
```

No conflict should occur.

---

# 138. Customer-Staff Same User Test

Same User linked to both:

```text
Staff
Customer
```

must preserve separate access semantics.

Internal Staff session must not accidentally expose Customer-only/internal CRM data beyond permissions.

Customer-facing context must not inherit Staff workspace permissions unless the user explicitly enters internal workspace through TenantMembership.

---

# 139. UI Implications

Internal Staff screens should use:

```text
StaffId
```

for operational records.

Account-management UI should use:

```text
UserId / account link state
```

Customer CRM screens use:

```text
CustomerId
```

Do not visually conflate them.

---

# 140. Search UX

Internal global "people" search may eventually aggregate:

```text
Staff
Customers
Users
```

but results must clearly label entity type.

This is a UI search concern.

The underlying domain entities remain separate.

---

# 141. Reporting

Business reports typically group by:

```text
StaffId
CustomerId
```

not UserId.

Authentication/security reports group by:

```text
UserId
```

Use the identity that matches the report domain.

---

# 142. Audit

Audit actor:

```text
UserId
```

Audit target:

```text
StaffId
CustomerId
AppointmentId
...
```

This separation is deliberate.

---

# 143. Database Indexes

Likely indexes:

Staff:

```text
(TenantId, UserId)
(TenantId, IsActive)
```

Customer:

```text
(TenantId, UserId)
(TenantId, NormalizedMobile)
(TenantId, IsActive)
```

Exact indexes depend on query patterns.

---

# 144. Database Constraints

Potential constraints:

```text
Staff.UserId nullable FK User
Customer.UserId nullable FK User

unique active Staff link per Tenant/User
unique active Customer link per Tenant/User
```

Provider-specific filtered indexes may be used where appropriate.

---

# 145. Tenant Isolation

Staff and Customer must always carry:

```text
TenantId
```

according to ADR-001.

User does not become Tenant-owned merely because it links to Tenant entities.

---

# 146. Subscription

Staff/Customer existence may later be affected by limits such as:

```text
max_staff
```

from ADR-006.

User account identity itself should not be deleted when a Tenant loses Staff entitlement.

---

# 147. Authorization

Internal workspace authorization uses:

```text
User
+
TenantMembership
+
Permission
+
Scope
```

Staff link contributes to Own-scope resolution.

Customer-facing authorization uses:

```text
User
+
Customer link
+
Own customer resource semantics
```

---

# 148. Authentication

ADR-002 remains unchanged.

OpenIddict `sub` identifies:

```text
User
```

not Staff or Customer.

This is the stable authentication identity.

---

# 149. Rejected Alternatives

Rejected:

```text
User = Staff
User = Customer
Staff = Customer
Every Staff must have login
Every Customer must have login
Every Customer is TenantMembership
One Person table for all identities in MVP
Use StaffId as auth subject
Use CustomerId as auth subject
Share Customer records across Tenants
Auto-link Customer to User by phone string only
Cascade-delete Staff/Customer when User is deleted
```

---

# 150. Consequences

## Positive

This model provides:

```text
Clean authentication boundary
Offline/non-login Staff support
Guest Customer support
Tenant-isolated CRM
Multi-Tenant User support
Customer login support
Staff login support
Own-scope authorization
Future marketplace compatibility
Independent lifecycles
```

## Negative

It introduces:

```text
duplicate profile/contact fields
linking workflows
more explicit IDs
relationship resolution
customer/staff account-link testing
```

These costs are accepted because the entities represent genuinely different domain concepts.

---

# 151. MVP Implementation Priority

For MVP:

```text
User
Staff.UserId?
Customer.UserId?
TenantMembership
BranchMembership
```

are enough.

Do not build immediately:

```text
customer account claim UI
complex account merge
global professional profile
marketplace identity
custom identity federation
```

unless required by active backlog.

---

# 152. Recommended Initial Schema

Conceptual:

```text
Users
    Id
    Phone
    Email
    PasswordHash
    IsActive
    ...

TenantMemberships
    TenantId
    UserId
    ...

BranchMemberships
    BranchId
    UserId
    ...

Staff
    Id
    TenantId
    UserId?
    ...

Customers
    Id
    TenantId
    UserId?
    ...

StaffBranches
    StaffId
    BranchId

StaffServices
    StaffId
    ServiceId
```

---

# 153. Final Decision Summary

Shinera separates:

```text
User
Staff
Customer
```

because they represent three different identities:

```text
User
→ authentication/account identity

Staff
→ Tenant-owned operational provider identity

Customer
→ Tenant-owned CRM/service-recipient identity
```

Both Staff and Customer may exist without a User.

Both may optionally link to a User.

A User may be linked to:

```text
multiple Staff records across different Tenants
multiple Customer records across different Tenants
Staff and Customer in the same Tenant
```

Internal workspace access requires:

```text
User + TenantMembership + Permission
```

and is not granted merely by Staff linkage.

Customer relationship does **not** use `TenantMembership` by default.

Customer-facing access uses:

```text
User + linked Customer + Own semantics
```

Staff Own-scope access uses:

```text
User + linked Staff
```

OpenIddict continues to authenticate `User` as the stable subject.

Tenant-specific Staff and Customer data remain isolated and never become global merely because they link to the same User.

This model preserves clean boundaries between:

```text
Authentication
Workforce
CRM
Authorization
Tenancy
```

while still supporting one real-world person playing multiple roles inside Shinera.
