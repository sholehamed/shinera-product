# Shinera — Development Status

**Project:** Shinera  
**Document Role:** Current implementation and delivery status  
**Last Updated:** 2026-10-06  
**Status Source:** Verified GitHub repository state  
**Repository Sync:** Backend synchronized; frontend implementation repository not yet populated beyond baseline files

---

# 1. Purpose

This document answers:

> **Where is Shinera right now?**

Repository reality is authoritative for implementation status.

Use:

- `docs/product/SHINERA-PRD.md` for product requirements.
- `docs/product/SHINERA-PRODUCT-BACKLOG.md` for delivery scope.
- `docs/development/PROJECT-INSTRUCTIONS.md` for engineering rules.
- `docs/development/DEFINITION-OF-DONE.md` for completion gates.
- `docs/architecture/decisions/` for approved architecture decisions.
- this document for the current verified implementation state.

---

# 2. Repository Status

Shinera uses three repositories:

```text
sholehamed/shinera-product
sholehamed/shinera-backend
sholehamed/shinera-frontend
```

Current state:

| Repository | State |
|---|---|
| `shinera-product` | Product, development, architecture and ADR documents are present |
| `shinera-backend` | Active .NET backend implementation through M0.8 is merged to `main` |
| `shinera-frontend` | Repository exists but currently contains only baseline README/gitignore files |

---

# 3. Backend Milestone Snapshot

The backend has completed the following merged milestones:

| Milestone | Status | Scope |
|---|---|---|
| M0.1 — Backend Foundation Alignment | `Done` | Solution/foundation alignment and baseline engineering conventions |
| M0.2 — Branch & Workspace Foundation | `Done` | Tenant/branch workspace foundation |
| M0.3 — Registration & Business Profile Foundation | `Done` | Registration transaction and business profile foundation |
| M0.4 — Service Catalog Foundation | `Done` | Service category/service catalog foundation |
| M0.5 — Staff & Workforce Foundation | `Done` | Staff and workforce assignment foundation |
| M0.6 — Weekly Schedule Foundation | `Done` | Weekly schedule, day-off and break foundation |
| M0.7 — Customer CRM Foundation | `Done` | Customer CRM and notes foundation |
| M0.8 — Appointment Core Foundation | `Done` | Appointment entity/state, create flow, availability and conflict detection |

A corrective ADR compliance pass covering M0.2–M0.8 was merged after M0.8.

Backend corrective PR:

```text
PR #9 — ADR compliance corrective pass — M0.2 to M0.8
Merged: 2026-10-06
Merge commit: 57a53a37ac77341d95c4833f8096df95c609cb07
```

---

# 4. Current Backend Capability

The backend currently includes meaningful vertical foundations for:

```text
Platform/Foundation
Authentication / OpenIddict foundation
Tenant / Workspace
Branch
Registration
Business Profile
Subscription / Feature Gating foundation
Authorization / Permission model
Service Catalog
Staff / Workforce
Weekly Staff Schedule
Customer CRM
Appointment Core
```

Appointment Core currently includes:

```text
Appointment entity
Appointment status model
Create Appointment
Available Time Slots
Conflict Detection
Staff schedule / break evaluation
Staff-to-Branch eligibility
Staff-to-Service eligibility
Tenant and Branch authorization
UTC appointment snapshots
Branch timezone semantics
Database-backed concurrency protection
```

---

# 5. ADR Compliance Status

A repository-level audit was performed against ADR-001 through ADR-008.

| ADR | Status | Current Result |
|---|---|---|
| ADR-001 Multi-Tenancy Strategy | `Compliant for current scope` | No critical corrective deviation found |
| ADR-002 Authentication with OpenIddict | `Implementation aligned / verification debt remains` | No corrective architectural deviation identified in M0.2–M0.8; deeper protocol integration coverage remains an auth-lifecycle task |
| ADR-003 Authorization and Permission Model | `Compliant for current scope` | Dynamic permission/scope model preserved |
| ADR-004 Appointment Concurrency Strategy | `Corrected and verified` | Pessimistic Staff booking lock, final revalidation, UTC overlap and real concurrent SQL Server test implemented |
| ADR-005 Date and Time Strategy | `Corrected and verified` | IANA timezone model, Tenant/Branch timezone, UTC snapshots and DST validation implemented |
| ADR-006 Subscription and Feature Gating | `Compliant for current scope` | No critical corrective deviation found |
| ADR-007 User / Staff / Customer Identity Model | `Compliant for current scope` | Separation preserved |
| ADR-008 Repository and Deployment Separation | `Compliant` | Product/backend/frontend repositories remain separated |

Detailed audit:

```text
docs/development/ADR-COMPLIANCE-AUDIT-M0.2-M0.8.md
```

---

# 6. Appointment Concurrency State

Appointment booking now follows the approved ADR-004 authority path:

```text
Begin Transaction
→ Acquire Tenant + Staff pessimistic DB lock
→ Revalidate booking inputs
→ Re-evaluate schedule / eligibility
→ Check canonical UTC overlap
→ Insert Appointment
→ Commit
```

The same Staff booking lock is used by availability-affecting Workforce mutations:

```text
Staff active/inactive changes
Staff update affecting bookability
Staff-to-Branch replacement
Staff-to-Service replacement
Weekly Schedule / Break replacement
```

Blocking appointment states are centralized as:

```text
Pending
Confirmed
Upcoming
InProgress
```

Non-blocking states include:

```text
Completed
Cancelled
NoShow
```

Lock timeout/deadlock behavior is mapped to stable application conflict behavior rather than exposing raw SQL errors.

---

# 7. Date / Time State

The backend now follows ADR-005 for the implemented Appointment scope.

Implemented:

```text
Tenant.DefaultTimeZoneId
Branch.TimeZoneId
IANA timezone validation
Central timezone resolver
UTC conversion
DST invalid local time rejection
DST ambiguous local time rejection
Appointment.StartUtc
Appointment.EndUtc
Appointment.TimeZoneId snapshot
Local DateOnly / TimeOnly business semantics
Canonical UTC overlap comparison
```

Existing historical Appointment meaning is protected by persisted UTC/timezone snapshots rather than being reinterpreted if Branch timezone changes later.

---

# 8. Verification and CI

Latest ADR corrective backend validation:

```text
Workflow: backend-ci
Run: #227
Result: SUCCESS
```

Verified gates:

```text
Build                           PASS
Identity migration chain        PASS
Subscription migration chain    PASS
Services migration chain        PASS
Workforce migration chain       PASS
CRM migration chain             PASS
Appointments migration chain    PASS
Application.Tests               147 / 147 PASS
IntegrationTests                29 / 29 PASS
Total automated tests           176 / 176 PASS
```

The Integration suite includes a real SQL Server concurrent booking test:

```text
Two concurrent requests
Same Tenant
Same Staff
Same interval

Expected and verified:
exactly one succeeds
exactly one receives conflict
```

---

# 9. Deferred Appointment Scope

M0.8 intentionally does not complete the whole Appointment epic.

Still deferred:

```text
Appointment Details
Appointment List
Calendar View
Reschedule Appointment
Cancel Appointment
Start Service
Complete Service
No-show command
Special Schedule
Time Off
Branch working-hours overlay
Force Appointment / VIP force request
Appointment notifications
Payment integration
```

These should be implemented through later backlog stories while preserving ADR-004 and ADR-005 rules.

---

# 10. Frontend Status

The current `shinera-frontend` GitHub repository does not yet contain the active Angular implementation discussed previously.

Therefore frontend completion must not be inferred from prior local/project work.

Current GitHub-verifiable frontend state:

```text
Repository baseline exists
Production Angular source not yet synchronized
Backend timezone contract has no committed frontend consumer yet
```

Before marking frontend stories Done, synchronize the actual Angular project and validate:

```text
Angular build
Authentication client
Registration contract
Tenant / Branch timezone inputs
Permissions
Subscription behavior
Appointment flows
RTL / Persian presentation
E2E coverage
```

---

# 11. Current Product Position

Backend progression is now best described as:

```text
Foundation Ready
→ Workspace Ready foundation
→ Service Ready foundation
→ Workforce Ready foundation
→ CRM Ready foundation
→ Booking Core foundation
```

The product is **not yet full MVP complete**.

The next backend work should extend Appointment Core rather than reopening completed M0.1–M0.8 foundations unless a new defect or approved ADR change requires it.

---

# 12. Next Recommended Backend Sequence

Continue from M0.8 with the next product-approved Appointment capabilities.

Recommended order:

```text
1. Appointment Details
2. Appointment List
3. Calendar View
4. Reschedule
5. Cancel
6. Start Service
7. Complete Service
8. No-show
9. Special Schedule / Time Off integration
10. Branch working-hours overlay
```

Any Appointment mutation that creates, moves, cancels, or materially changes bookability must explicitly evaluate ADR-004 concurrency requirements.

Any date/time-sensitive story must explicitly evaluate ADR-005 temporal semantics.

---

# 13. Known Verification Debt

The ADR audit found no remaining blocker in the merged M0.2–M0.8 corrective scope.

Remaining non-blocking verification debt includes:

```text
ADR-002 deeper OpenIddict protocol-flow integration coverage
Frontend repository synchronization
Full Appointment epic completion
Frontend E2E coverage once Angular source is synchronized
```

These are tracked as future work and are not hidden implementation exceptions.

---

# 14. Verification Principle

Whenever this document conflicts with repository reality:

```text
Repository reality wins for implementation status.
```

Then this document must be updated.

Whenever implementation behavior conflicts with an approved ADR or product requirement:

```text
Do not silently change the requirement.
Audit the deviation and either correct implementation or explicitly revise the decision.
```
