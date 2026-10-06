# Shinera — ADR Compliance Audit M0.2–M0.8

**Audit Date:** 2026-10-06  
**Backend Repository:** `sholehamed/shinera-backend`  
**Product Repository:** `sholehamed/shinera-product`  
**Corrective Backend PR:** `#9 — ADR compliance corrective pass — M0.2 to M0.8`  
**Merged Commit:** `57a53a37ac77341d95c4833f8096df95c609cb07`

---

# 1. Purpose

This audit was performed after M0.8 to answer:

> Which backend implementation created through M0.2–M0.8 deviated from the approved Shinera ADRs, and what must be corrected before continuing development?

The review covered ADR-001 through ADR-008, with special attention to cross-cutting decisions that materially affect booking correctness, tenant isolation and temporal behavior.

---

# 2. Audit Outcome

Two material implementation gaps required corrective code:

```text
ADR-004 — Appointment Concurrency Strategy
ADR-005 — Date and Time Strategy
```

No critical corrective implementation deviation was identified for the current M0.2–M0.8 scope in:

```text
ADR-001 — Multi-Tenancy Strategy
ADR-002 — Authentication with OpenIddict
ADR-003 — Authorization and Permission Model
ADR-006 — Subscription and Feature Gating
ADR-007 — User / Staff / Customer Identity Model
ADR-008 — Repository and Deployment Separation
```

ADR-002 still has verification debt around complete protocol-flow integration testing. This is not the same as finding an architectural deviation in the M0.2–M0.8 implementation.

---

# 3. ADR-001 — Multi-Tenancy Strategy

**Result:** Compliant for current scope.

Verified architectural characteristics include:

```text
Explicit Tenant ownership
CurrentTenant-based filtering
Tenant-aware write protection
Branch-aware authorization
Cross-tenant validation
Tenant isolation tests
```

No corrective architecture change was required by this audit.

---

# 4. ADR-002 — Authentication with OpenIddict

**Result:** No corrective M0.2–M0.8 deviation identified.

The backend continues to use OpenIddict and preserves separation between authentication and application authorization.

The audit does not claim the entire ADR-002 DoD is complete.

Remaining verification debt includes deeper evidence for:

```text
Authorization Code flow
PKCE
Refresh behavior
Logout/revocation
Inactive-user behavior
Client permissions
Frontend refresh race handling
```

These belong to the authentication lifecycle scope and should be completed before Authentication is marked fully Done.

---

# 5. ADR-003 — Authorization and Permission Model

**Result:** Compliant for current scope.

Current implementation preserves:

```text
Backend-enforced authorization
Resource + action permission model
Tenant / Branch / Own scope behavior
Branch access validation
Permission catalog integration
Automated authorization checks
```

No corrective architectural rewrite was required.

---

# 6. ADR-004 — Appointment Concurrency Strategy

**Initial result:** Material deviation found.  
**Final result:** Corrected and verified.

## Gaps found

The initial Appointment implementation performed overlap checking under transaction isolation but did not yet implement the approved serialization resource:

```text
Tenant + Staff booking lock
```

Availability-affecting Workforce writes were also not coordinated on the same resource.

Blocking-state semantics also needed to be made explicit and canonical.

## Corrections

Added:

```text
IStaffBookingConcurrencyGuard
SqlServerStaffBookingConcurrencyGuard
SQL Server UPDLOCK + HOLDLOCK + ROWLOCK
TenantId + StaffId lock validation
Short transaction ownership in application handler
Final availability revalidation after lock acquisition
Canonical UTC overlap recheck
Stable lock-contention error mapping
```

Create Appointment now follows:

```text
Begin Transaction
→ Acquire Staff booking lock
→ Validate Customer
→ Revalidate Branch / Service / Staff / Schedule
→ Check UTC conflict
→ Insert Appointment
→ Commit
```

Availability-affecting Workforce writes now coordinate on the same lock:

```text
Update Staff
Set Staff Active
Replace Staff Branches
Replace Staff Services
Replace Weekly Schedule / Breaks
```

Canonical blocking states:

```text
Pending
Confirmed
Upcoming
InProgress
```

Non-blocking states:

```text
Completed
Cancelled
NoShow
```

## Error handling

SQL Server lock timeout/deadlock/provider contention is not exposed directly.

Application behavior maps booking contention to stable conflict semantics.

For Appointment creation:

```text
appointment.conflict
HTTP 409 behavior
```

## Verification

A real SQL Server integration test runs two genuinely concurrent booking requests for:

```text
same Tenant
same Staff
same interval
```

Verified result:

```text
exactly one succeeds
exactly one conflicts
```

This is the required core evidence for the ADR-004 serialization strategy.

---

# 7. ADR-005 — Date and Time Strategy

**Initial result:** Material deviation found.  
**Final result:** Corrected and verified.

## Gaps found

The initial booking foundation used local `DateOnly + TimeOnly` values without the complete approved timezone/UTC persistence model.

This was insufficient for:

```text
multi-timezone Branch support
safe Appointment instant comparison
DST behavior
historical timezone stability
```

## Corrections

Added:

```text
ITimeZoneResolver
TimeZoneResolver
Tenant.DefaultTimeZoneId
Branch.TimeZoneId
IANA timezone validation
Appointment.StartUtc
Appointment.EndUtc
Appointment.TimeZoneId snapshot
UTC-based overlap comparison
```

Registration captures Tenant timezone and Main Branch inherits it unless explicitly overridden.

Branch create/update validates IANA identifiers.

Appointment validation converts Branch-local date/time into canonical UTC instants.

Service duration is applied to the instant and converted back to Branch-local time for schedule-boundary validation.

## DST behavior

Explicit tests verify:

```text
valid IANA conversion
round-trip conversion
nonexistent DST local time rejection
ambiguous DST local time rejection
fixed numeric offset rejection
unknown timezone rejection
```

The implementation does not silently shift nonexistent local times or arbitrarily choose an ambiguous DST offset.

## Historical semantics

Appointments persist:

```text
business Date
business StartTime / EndTime
StartUtc / EndUtc
TimeZoneId snapshot
```

This protects the original booking meaning if Branch timezone configuration changes later.

---

# 8. ADR-006 — Subscription and Feature Gating

**Result:** No critical corrective deviation found in current scope.

Backend feature entitlement architecture remains separate from permission authorization.

No plan-name-driven corrective rewrite was required by this audit.

---

# 9. ADR-007 — User / Staff / Customer Identity Model

**Result:** Compliant for current scope.

The model continues to keep:

```text
User
Staff
Customer
```

as separate domain concepts.

No corrective merge of these identities was found or required.

---

# 10. ADR-008 — Repository and Deployment Separation

**Result:** Compliant.

Current repositories remain separated:

```text
shinera-product
shinera-backend
shinera-frontend
```

The backend corrective pass was implemented only in `shinera-backend`.

Architecture and implementation-status synchronization is maintained in `shinera-product`.

The frontend repository currently contains only baseline files; therefore no frontend contract migration was required by this corrective pass.

---

# 11. Database and Migration Verification

The corrective pass added and validated:

```text
Identity timezone migration
Appointments UTC/timezone migration
Model snapshots
UTC range constraint
UTC conflict index
```

CI verified all migration chains:

```text
Identity        PASS
Subscription    PASS
Services        PASS
Workforce       PASS
CRM             PASS
Appointments    PASS
```

---

# 12. Automated Test Result

Final corrective CI:

```text
backend-ci #227
SUCCESS
```

Automated test result:

```text
Application.Tests     147 passed
IntegrationTests       29 passed
Total                 176 passed
Failures                0
```

The test suite includes temporal/DST regression coverage and real-database booking concurrency coverage.

---

# 13. Milestone Impact

Audit impact by milestone:

| Milestone | Audit Result |
|---|---|
| M0.2 Branch & Workspace | Timezone model corrected |
| M0.3 Registration & Business Profile | Registration timezone contract corrected |
| M0.4 Service Catalog | No material ADR correction required |
| M0.5 Staff & Workforce | Booking-affecting Staff mutations serialized |
| M0.6 Weekly Schedule | Schedule/Break mutation serialized |
| M0.7 Customer CRM | No material ADR correction required |
| M0.8 Appointment Core | Concurrency, UTC interval, timezone/DST and blocking-state behavior corrected |

---

# 14. Final Assessment

For the merged M0.2–M0.8 backend scope:

```text
No known unresolved ADR-004 or ADR-005 blocker remains.
```

The corrective backend PR passed build, migration and automated test gates before merge.

Future work must continue treating approved ADRs as implementation constraints, especially:

```text
Appointment mutation → ADR-004
Date/time-sensitive behavior → ADR-005
Tenant-owned behavior → ADR-001
Protected behavior → ADR-003
Feature-gated behavior → ADR-006
```

If future implementation conflicts with an ADR, the conflict must be surfaced explicitly. Do not silently implement a different architecture.
