# ADR-004 — Appointment Concurrency Strategy

**Project:** Shinera  
**Status:** Accepted  
**Date:** 2026-10-05  
**Decision Type:** Architecture / Data Consistency / Concurrency  
**Scope:** Appointments, Availability, Staff Schedule, Database, Transactions, Integration Tests  
**Related Documents:**
- `SHINERA-PRD.md`
- `SHINERA-PRODUCT-BACKLOG.md`
- `PROJECT-INSTRUCTIONS.md`
- `ARCHITECTURE-OVERVIEW.md`
- `DEFINITION-OF-DONE.md`
- `ADR-001-MULTI-TENANCY-STRATEGY.md`
- `ADR-002-AUTHENTICATION-WITH-OPENIDDICT.md`
- `ADR-003-AUTHORIZATION-AND-PERMISSION-MODEL.md`

---

# 1. Context

Appointment booking is the core transactional workflow of Shinera.

The system must guarantee that two concurrent requests cannot successfully reserve overlapping time for the same Staff member.

A simple implementation such as:

```text
Check availability
↓
Slot appears free
↓
Insert appointment
```

is unsafe under concurrency.

Example race:

```text
Request A checks 10:00 → free
Request B checks 10:00 → free

Request A inserts 10:00
Request B inserts 10:00

Result:
Double booking
```

Both requests may be individually correct while the final database state is invalid.

This problem exists even if:

```text
frontend disables occupied slots
API checks overlap
EF Core tracks entities
Appointment has a concurrency token
```

because the conflict may involve two new rows where no conflicting Appointment row existed when either request performed its read.

---

# 2. Decision

Shinera will protect appointment booking with:

```text
Short Database Transaction
+
Pessimistic Lock on the Staff booking resource
+
Availability Revalidation inside the lock
+
Atomic Appointment Write
```

The Staff record is the initial concurrency serialization resource.

Conceptually:

```text
Begin Transaction
↓
Acquire exclusive/update lock for Staff
↓
Revalidate Staff bookability
↓
Recalculate/check effective availability
↓
Query blocking overlapping Appointments
↓
If conflict → return business conflict
↓
Insert/Update Appointment
↓
SaveChanges
↓
Commit
```

The Staff lock remains held until the transaction commits or rolls back.

This serializes concurrent booking mutations for the same Staff member.

---

# 3. Core Guarantee

For a single Staff member:

```text
Only one availability-affecting write operation
may pass the final availability check at a time.
```

Therefore two concurrent requests for overlapping intervals cannot both commit successfully when all appointment-mutating paths follow this ADR.

---

# 4. Why Staff Is the Lock Resource

An Appointment consumes Staff availability.

The Staff entity is:

```text
stable
already persistent
tenant-owned
always present before booking
naturally shared by all appointments for that specialist
```

Using Staff as the lock resource avoids requiring an artificial lock row during MVP.

Example:

```text
Staff Sara
    ↓ lock
All booking changes affecting Sara
```

While one booking transaction holds Sara's booking lock, another booking mutation for Sara waits until the first transaction completes.

---

# 5. Lock Granularity

Initial lock granularity is:

```text
Tenant + Staff
```

not:

```text
Tenant + Staff + Date
```

This means concurrent appointment writes for the same Staff on different dates may briefly serialize.

This is intentionally accepted for MVP because:

- booking transactions are short,
- salon booking throughput per individual Staff member is expected to be moderate,
- implementation remains simple and auditable,
- correctness is more important than theoretical maximum write throughput.

If profiling later proves this too coarse, the strategy may evolve to a per-Staff/per-business-date lock resource.

That would require an ADR update.

---

# 6. Database Independence

Shinera must not make the core Appointment domain dependent on one database-specific locking syntax.

Application code should depend on an abstraction such as:

```text
IAppointmentConcurrencyGuard
```

or:

```text
IStaffBookingLock
```

Exact interface naming follows repository conventions.

Infrastructure provides the provider-specific lock implementation.

Conceptually:

```text
Application
↓
Appointment Concurrency Abstraction
↓
Infrastructure
├── SQL Server locking implementation
└── PostgreSQL locking implementation
```

---

# 7. SQL Server Implementation

For SQL Server, the infrastructure implementation may acquire an update/exclusive lock on the Staff row within the active transaction using appropriate SQL Server locking semantics.

Conceptual technique:

```text
UPDLOCK
+
HOLDLOCK / transaction-held lock
```

or an equivalent safe provider-specific statement.

The purpose is:

```text
Lock this Staff booking resource
until transaction completion.
```

Exact SQL must be integration-tested against the selected SQL Server version/provider.

---

# 8. PostgreSQL Implementation

For PostgreSQL, the infrastructure implementation may use:

```text
SELECT ... FOR UPDATE
```

on the Staff row inside the transaction.

The lock is held until:

```text
COMMIT
or
ROLLBACK
```

Exact SQL/provider behavior must be integration-tested.

---

# 9. Default Transaction Isolation

The booking strategy does not require making every application transaction:

```text
SERIALIZABLE
```

by default.

The preferred model is:

```text
normal short transaction
+
explicit Staff booking lock
```

This narrows contention to the resource whose schedule is being modified.

Higher transaction isolation may still be used for specific operations if later evidence requires it.

---

# 10. Why Global SERIALIZABLE Is Not the Default

`SERIALIZABLE` provides strong guarantees, but it may introduce:

```text
broader blocking
range locks
serialization failures
higher retry requirements
database-specific behavior
```

Using it globally for all Appointment operations would make concurrency behavior harder to predict and potentially reduce throughput.

Shinera instead makes the serialization point explicit:

```text
Staff booking resource
```

---

# 11. Why Optimistic Concurrency Alone Is Rejected

EF Core optimistic concurrency tokens are useful when two requests modify the same existing row.

Example:

```text
Appointment A loaded with Version 4

Request 1 updates Appointment A
Request 2 updates Appointment A

Version mismatch
→ concurrency conflict
```

But this does not solve:

```text
No Appointment exists at 10:00

Request A inserts new Appointment
Request B inserts new Appointment
```

Both inserts may target different rows.

Therefore:

```text
Appointment RowVersion / concurrency token
```

is useful but insufficient for slot reservation.

---

# 12. Optimistic Concurrency Still Has a Role

Shinera should use optimistic concurrency for edits to an existing Appointment where appropriate.

Conceptually:

```text
Appointment
└── Version / concurrency token
```

This protects against lost updates such as:

```text
Receptionist reschedules appointment
while
Owner cancels same appointment
```

The final architecture is therefore:

```text
Pessimistic Staff booking lock
→ protects schedule capacity / overlapping intervals

Appointment optimistic concurrency token
→ protects concurrent modification of same Appointment row
```

These solve different problems.

---

# 13. Why Unique Start-Time Constraint Alone Is Rejected

A unique index such as:

```text
UNIQUE(TenantId, StaffId, StartTime)
```

would prevent:

```text
10:00–11:00
10:00–10:30
```

from coexisting.

But it would not prevent:

```text
10:00–11:00
10:30–11:30
```

because the start times differ.

Therefore exact start-time uniqueness is not sufficient for interval overlap protection.

It may be used only as optional defense in depth if useful.

---

# 14. Why Database-Specific Interval Constraints Are Not the Primary Strategy

Some databases provide powerful provider-specific techniques for interval exclusion.

Using such a feature as the sole architecture would tightly bind Appointment correctness to one provider.

Shinera currently preserves database-provider flexibility.

Therefore:

```text
provider-specific interval exclusion
```

is not the primary MVP concurrency strategy.

It may later be added as additional defense in depth for a selected production provider.

---

# 15. Appointment Interval Semantics

Appointments use half-open time intervals:

```text
[Start, End)
```

Meaning:

```text
Start is inclusive
End is exclusive
```

Example:

```text
Appointment A: 10:00–11:00
Appointment B: 11:00–12:00
```

These are adjacent, not overlapping.

---

# 16. Overlap Formula

Two intervals overlap when:

```text
existing.Start < requested.End
AND
requested.Start < existing.End
```

Equivalent conceptual predicate:

```csharp
existing.Start < requestedEnd
&& requestedStart < existing.End
```

Do not use only:

```text
same StartTime
```

or:

```text
requested Start between existing Start/End
```

because those fail for some interval combinations.

---

# 17. Blocking Appointment Statuses

Only Appointment states that occupy Staff time should participate in availability conflict checks.

Initial conceptual blocking set:

```text
Pending
Confirmed
Upcoming
InProgress
```

Historical/non-reserving states such as:

```text
Cancelled
NoShow
```

must not block future availability.

`Completed` represents historical consumed time and naturally remains part of historical schedule data; for past intervals it does not affect future availability.

Exact status semantics must remain aligned with the Appointment state model.

---

# 18. Pending Reservation Semantics

If `Pending` means the slot has already been reserved, it blocks availability.

If a future product feature introduces non-binding booking requests, they must use a different state/model rather than overloading `Pending`.

The concurrency engine must clearly know which records consume capacity.

---

# 19. Appointment Duration

The interval used for concurrency must come from the Appointment's effective duration at creation/reschedule time.

Do not depend on the current Service duration for historical appointments after booking.

Appointment should persist enough timing information to preserve the booked interval, such as:

```text
Start
End
```

or:

```text
Start
Duration snapshot
```

Preferred availability queries should be able to determine:

```text
Start
End
```

without needing the current mutable Service definition.

---

# 20. Create Appointment Transaction

Required sequence:

```text
Validate authentication/authorization context
↓
Begin transaction
↓
Acquire Staff booking lock
↓
Reload/revalidate relevant Staff state
↓
Validate Tenant/Branch ownership
↓
Validate Staff active
↓
Validate Service active
↓
Validate Staff-Service assignment
↓
Validate Staff-Branch assignment
↓
Resolve effective schedule
↓
Apply breaks/time off/special schedule
↓
Calculate requested interval
↓
Query blocking overlapping Appointments
↓
Conflict?
    yes → rollback + business error
    no
↓
Create Appointment
↓
SaveChanges
↓
Commit
```

The final conflict check must occur after acquiring the Staff lock.

---

# 21. Pre-Validation Outside the Transaction

Cheap validation may occur before opening the transaction to reduce lock time.

Examples:

```text
request format
required fields
basic date validation
permission key existence
```

However:

```text
availability
Staff status
schedule
assignments
overlap
```

must be revalidated under the concurrency lock before commit if they affect correctness.

Pre-validation is an optimization, not the final guarantee.

---

# 22. Keep Transactions Short

The transaction must not include:

```text
user interaction
network calls to external providers
SMS sending
email sending
payment-provider calls
long-running reporting
```

The transaction should contain only the database work required to guarantee booking consistency.

Conceptual:

```text
Lock
Validate
Write
Commit
```

Then perform noncritical side effects after commit.

---

# 23. Notifications After Commit

Do not send an appointment-created notification before the booking transaction commits.

Preferred:

```text
Commit Appointment
↓
Publish/queue notification work
```

This prevents:

```text
Customer receives confirmation
but database transaction rolls back
```

A future Outbox pattern may be introduced if reliable event delivery becomes required.

---

# 24. Available Slots Are Advisory

The available-slots endpoint provides a point-in-time view.

Flow:

```text
GET Available Slots
→ 10:00 appears free

User waits

Another user books 10:00

Original user submits 10:00
```

The original request must then fail safely.

Therefore:

```text
Displayed available slot != reservation
```

Only successful Appointment creation reserves time.

---

# 25. Available Slots Endpoint

The read-only availability endpoint does not need to acquire the booking lock for the entire user interaction.

It computes:

```text
best current availability snapshot
```

The create/reschedule command performs the authoritative re-check.

This avoids holding database locks while a user chooses a time.

---

# 26. Conflict Response

If the requested interval becomes unavailable before commit:

Return a business conflict.

Recommended error code:

```text
appointment.conflict
```

Recommended HTTP status:

```text
409 Conflict
```

Frontend behavior:

```text
Show slot no longer available
↓
Refresh available slots
↓
Allow user to choose another time
```

This is an expected business condition, not an unexpected exception.

---

# 27. Lock Wait vs Business Conflict

Two different conditions exist.

## Lock Wait

Another transaction is currently modifying the Staff booking schedule.

Expected behavior:

```text
wait briefly for lock
↓
acquire lock
↓
re-check availability
```

## Appointment Conflict

After lock acquisition:

```text
requested interval is already occupied
```

Return:

```text
409 appointment.conflict
```

Do not return a concurrency/database exception directly to the frontend.

---

# 28. Lock Timeout

Database lock waits must not be allowed to hang indefinitely.

Use provider-appropriate:

```text
command timeout
lock timeout
cancellation token
```

If lock acquisition fails due to timeout/transient contention:

Return or translate to a stable application error such as:

```text
appointment.booking_busy
```

or retry internally when safe.

Exact retry policy is implementation-level and should remain bounded.

---

# 29. Retry Policy

Retries are appropriate only for transient technical concurrency failures.

Examples:

```text
deadlock victim
serialization failure if stricter isolation is used
transient connection failure
```

Do not automatically retry:

```text
appointment.conflict
invalid schedule
permission denied
feature unavailable
```

because those are business results.

---

# 30. Retry Must Re-run the Whole Transaction

If a transient concurrency retry occurs:

```text
Acquire Lock
↓
Revalidate
↓
Check overlap
↓
Write
```

must all run again.

Do not retry only `SaveChanges()` using stale availability assumptions.

---

# 31. EF Core Execution Strategy

When the selected EF Core provider uses retrying execution strategies, manually controlled transactions must be executed using the provider-supported execution-strategy pattern.

Do not combine:

```text
manual transaction
+
automatic retry
```

incorrectly.

The entire transactional delegate must be retryable as one unit.

---

# 32. Reschedule Appointment

Rescheduling is equivalent to:

```text
release old interval
+
reserve new interval
```

but it must be atomic.

Conceptual sequence:

```text
Begin transaction
↓
Acquire required Staff lock(s)
↓
Load Appointment
↓
Validate current Appointment version/state
↓
Validate new schedule/availability
↓
Check overlap excluding current Appointment
↓
Update interval/staff/branch
↓
Save
↓
Commit
```

---

# 33. Reschedule to Same Staff

If Staff does not change:

```text
Acquire one Staff booking lock
```

Then conflict check excludes:

```text
current Appointment.Id
```

from the overlap query.

---

# 34. Reschedule to Different Staff

If moving:

```text
Staff A
→
Staff B
```

both booking resources may be involved.

Acquire both Staff locks before mutation.

---

# 35. Deterministic Lock Ordering

Whenever an operation acquires more than one Staff lock:

```text
sort lock keys deterministically
```

Example:

```text
ascending StaffId
```

Then acquire in that order.

This reduces deadlock risk.

Never acquire:

```text
A then B
```

in one path and:

```text
B then A
```

in another path when both resources are involved.

---

# 36. Multi-Resource Future

Future scheduling may introduce resources such as:

```text
Room
Chair
Equipment
Machine
```

If a booking consumes multiple resources, the same strategy may evolve into:

```text
Acquire all required booking-resource locks
in deterministic order
↓
Validate availability
↓
Commit reservation
```

This ADR currently guarantees Staff availability only.

Additional resource capacity requires a deliberate extension.

---

# 37. Cancellation

Cancellation changes whether a future interval consumes capacity.

Preferred flow:

```text
Begin transaction
↓
Acquire Staff booking lock
↓
Load Appointment
↓
Validate state/version
↓
Cancel
↓
Save
↓
Commit
```

This guarantees that booking operations observe a consistent ordering between:

```text
slot release
and
new reservation
```

---

# 38. No-Show

A NoShow transition should use the same booking-mutation discipline if it changes whether an interval is considered blocking.

For normal past appointments this has little contention, but consistency is preferred.

---

# 39. Start / Complete

Starting or completing an Appointment generally does not change the originally booked interval.

These transitions primarily require:

```text
Appointment optimistic concurrency
+
state-machine validation
```

A Staff booking lock is not strictly required unless the operation also changes availability semantics.

Implementation may avoid the booking lock for pure status transitions that do not free/change the interval.

---

# 40. Appointment Deletion

Appointments should generally not be hard-deleted as a routine booking operation.

Cancellation/state transition preserves:

```text
history
audit
payment relationship
customer history
```

If administrative deletion is later supported, it must respect booking concurrency and audit rules.

---

# 41. Schedule Changes

A Staff schedule change can race with Appointment creation.

Example:

```text
Request A books 18:00
Request B changes Staff end time to 17:00
```

If the two paths use unrelated concurrency rules, the system may produce an Appointment outside the final schedule.

Therefore writes that change Staff bookability must coordinate on the same Staff booking lock.

---

# 42. Availability-Affecting Staff Changes

Operations that should use the Staff booking concurrency guard when they can invalidate booking rules include:

```text
Staff activation/deactivation
Staff-Branch assignment changes
Staff-Service assignment changes
Weekly schedule changes
Break changes
Special schedule changes
Time-off changes
```

The exact command may additionally define rules for existing future Appointments.

---

# 43. Existing Appointments During Schedule Change

Changing a schedule may conflict with already-booked future Appointments.

This is a product/business decision separate from raw concurrency.

Possible future policies include:

```text
reject schedule change
warn and require resolution
allow with explicit override
reschedule affected appointments
```

This ADR only requires that concurrency not produce an accidental inconsistent race.

The final business policy should be documented before implementation.

---

# 44. Staff Deactivation

Deactivating Staff while future Appointments exist requires a business rule.

Concurrency requirement:

```text
Staff deactivation
and
new Appointment booking
```

must not race.

Therefore Staff deactivation should acquire the same Staff booking lock before final validation/write.

---

# 45. Service Deactivation

Service deactivation affects whether new Appointments may be created.

Because the lock resource is Staff, a concurrent Service deactivation could theoretically race with booking.

The create Appointment transaction must revalidate Service active state close to the final write.

If Service lifecycle operations require strict simultaneous exclusion across many Staff, a broader strategy may be introduced later.

For MVP, the final in-transaction revalidation plus database transaction is the required baseline.

---

# 46. Staff-Service Assignment Changes

Removing a Staff-Service assignment affects bookability.

Such changes should coordinate with the Staff booking lock.

This prevents:

```text
booking validates assignment
while
assignment is concurrently removed
```

from committing in an undefined order.

---

# 47. Staff-Branch Assignment Changes

The same rule applies to Staff-Branch assignment.

Any operation that changes whether Staff may work in a Branch should coordinate using the Staff booking lock.

---

# 48. Appointment Query Index

Overlap checks must be efficient.

A likely index should support queries beginning with:

```text
TenantId
StaffId
```

plus time/date/status filtering.

Conceptual examples:

```text
(TenantId, StaffId, Date, StartTime)
```

or a timestamp-based equivalent.

Exact index design depends on the final temporal schema and production query plans.

---

# 49. Date Partition of Search

If Appointment stores:

```text
BusinessDate
StartTime
EndTime
```

overlap queries should first narrow by:

```text
TenantId
StaffId
BusinessDate
```

where cross-midnight appointments are not supported.

If cross-midnight appointments become supported, the interval model must account for that explicitly.

---

# 50. Cross-Midnight Appointments

MVP should avoid implicit support for appointments spanning business dates unless product requirements explicitly require it.

If unsupported:

```text
End must be within the supported business-day boundary
```

This simplifies:

```text
schedule evaluation
locking model
date filtering
reporting
```

If later supported, Appointment temporal design requires review.

---

# 51. Time Zone

Concurrency compares normalized domain time values, not formatted Persian date strings.

Date/time strategy follows the future:

```text
ADR-005 — Date and Time Strategy
```

The booking lock key and overlap predicate must use canonical server/domain values.

---

# 52. Tenant Isolation

The Staff booking lock must be tenant-safe.

Infrastructure lock query must include/validate:

```text
TenantId
StaffId
```

A Staff from another Tenant must not become lockable/usable through an arbitrary ID.

Concurrency protection does not replace Tenant authorization.

---

# 53. Branch Isolation

Appointment creation must validate:

```text
Branch belongs to Tenant
Staff may operate in Branch
User may access Branch
```

The Staff lock is not proof of Branch authorization.

---

# 54. Authorization

Required permission is evaluated before protected booking operations.

Examples:

```text
appointments.create
appointments.reschedule
appointments.cancel
```

Concurrency lock acquisition should not grant access.

Authorization follows ADR-003.

---

# 55. Public Booking

Public booking uses the same final booking command/concurrency engine as internal booking.

Do not implement:

```text
PublicAppointmentCreate
```

with a separate unsafe overlap algorithm.

Flow:

```text
Public request
↓
Resolve trusted Tenant
↓
Validate public booking rules
↓
Common Appointment booking workflow
↓
Staff lock
↓
Final availability check
↓
Commit
```

---

# 56. Any Available Staff

For:

```text
Any Available Staff
```

the system may first discover candidate Staff without locking.

Then:

```text
choose candidate
↓
acquire that Staff lock
↓
revalidate
```

If candidate became unavailable:

```text
try another candidate
```

using a bounded deterministic strategy.

Do not lock every Staff member merely to display available options.

---

# 57. Fairness

The database lock establishes serialization, not business fairness.

Shinera does not guarantee:

```text
first browser click always wins
```

It guarantees:

```text
only a valid booking commits
```

Database scheduling may determine which concurrent transaction acquires the lock first.

This is acceptable.

---

# 58. Idempotency

Concurrency and idempotency solve different problems.

A client may accidentally retry the same create request due to:

```text
network timeout
double click
reverse proxy retry
```

The Staff lock prevents overlap but may still allow duplicate logical requests if the second request is for a non-conflicting scenario or the first response was lost.

For public/payment-sensitive flows, Shinera may add:

```text
Idempotency-Key
```

support later.

Idempotency is not required by this ADR for all MVP internal appointment commands.

---

# 59. Duplicate Submit UX

Frontend should prevent obvious duplicate submission:

```text
disable submit while request pending
```

This improves UX but is not a concurrency guarantee.

Backend correctness must remain independent of frontend behavior.

---

# 60. Transaction Boundaries

The transaction begins as late as practical and ends as early as practical.

Good:

```text
basic request validation
↓
BEGIN
↓
lock
↓
critical validation
↓
write
↓
COMMIT
↓
notification
```

Bad:

```text
BEGIN
↓
call SMS provider
↓
wait for user/external system
↓
generate report
↓
write
↓
COMMIT
```

---

# 61. SaveChanges

The appointment write and all database changes that define the reservation must commit atomically.

Examples:

```text
Appointment row
Appointment audit state required for integrity
required booking metadata
```

Noncritical external effects belong after commit.

---

# 62. Audit

Appointment audit should preserve:

```text
Created
Rescheduled
Cancelled
Completed
NoShow
```

Concurrency failure is generally an operational/business conflict, not a successful audit event.

It may be logged/observed for metrics.

---

# 63. Observability

Useful booking telemetry:

```text
TenantId
BranchId
StaffId
AppointmentId
RequestedStart
RequestedEnd
Result
Conflict
LockWaitDuration
TransactionDuration
TraceId
```

Do not log sensitive customer data unnecessarily.

---

# 64. Metrics

Potential future metrics:

```text
appointment_conflict_count
appointment_booking_lock_wait
appointment_booking_duration
appointment_booking_retry_count
appointment_booking_timeout_count
```

These metrics can reveal whether Staff-level locking becomes a scalability bottleneck.

---

# 65. Performance Threshold for Revisit

The Staff-level lock strategy should be revisited only when evidence shows meaningful contention.

Possible signals:

```text
frequent lock waits
booking latency degradation
high timeout rate
large concurrent booking volume per Staff
```

Do not prematurely introduce more complex lock partitioning.

---

# 66. Future Per-Staff/Date Lock

If Staff-level locking becomes too coarse, the next likely evolution is:

```text
AppointmentBookingLock
├── TenantId
├── StaffId
└── BusinessDate
```

Composite key:

```text
(TenantId, StaffId, BusinessDate)
```

Booking operations would lock this row.

This allows concurrent bookings for the same Staff on different dates.

This optimization is deferred.

---

# 67. Why Lock Table Is Deferred

A dedicated lock table introduces:

```text
row lifecycle
missing-row creation race
cleanup policy
additional schema
more infrastructure code
```

Staff-level locking already provides correctness with less complexity.

Therefore lock-table optimization is not part of MVP.

---

# 68. Distributed Application Instances

The concurrency strategy must work when Shinera runs on multiple backend instances.

Database locking satisfies this because the database is the shared coordination point.

Rejected as sole mechanism:

```text
C# lock
SemaphoreSlim
static dictionary lock
in-memory mutex
```

These protect only one process.

---

# 69. Distributed Cache Lock

Redis/distributed locks are not required for appointment booking correctness in MVP.

The relational database already owns the transactional Appointment state and is the most direct serialization authority.

Adding distributed locking would create an additional failure mode.

---

# 70. Application-Level `lock`

Rejected:

```csharp
lock (...)
{
    // create appointment
}
```

Reasons:

- only protects one process,
- fails across replicas,
- fails after restart,
- not transactionally tied to database commit.

---

# 71. Optimistic Retry-Only Strategy

Rejected as primary strategy.

A design that:

```text
checks overlap
attempts insert
hopes a later concurrency token detects it
```

does not reliably detect interval conflicts between distinct inserted rows.

---

# 72. Table Locking

Locking the entire Appointment table is rejected.

It would serialize unrelated appointments across:

```text
different Staff
different Tenants
different dates
```

and cause unnecessary contention.

---

# 73. Tenant-Level Locking

Locking the entire Tenant for each booking is rejected.

Bookings for unrelated Staff inside the same salon must proceed concurrently.

The natural initial unit is:

```text
Staff
```

---

# 74. Branch-Level Locking

Locking an entire Branch is also rejected.

Different Staff in the same Branch can safely receive bookings concurrently.

---

# 75. Exact Database Constraint Defense

Where the selected database permits a safe additional constraint that catches some invalid states without creating portability problems, it may be added as defense in depth.

But:

```text
Application correctness must not rely on a partial constraint that fails to cover interval overlap.
```

---

# 76. Appointment Version

Appointment should include an application/provider-appropriate concurrency token for mutable existing-row operations.

Possible forms:

```text
rowversion
xmin/provider feature
application-managed Guid/version
```

Provider-independent domain surface should avoid leaking database-specific types where practical.

Exact implementation belongs to persistence configuration.

---

# 77. Concurrency Error Mapping

Existing-row optimistic concurrency failure should map to a stable business/application error.

Possible:

```text
appointment.concurrent_update
```

HTTP:

```text
409 Conflict
```

Frontend should refresh Appointment data and ask the user to retry the intended action if needed.

---

# 78. Booking Conflict vs Concurrent Update

Keep errors distinct.

```text
appointment.conflict
```

means:

```text
requested time is unavailable
```

```text
appointment.concurrent_update
```

means:

```text
same Appointment changed since client/app operation began
```

This distinction improves UX and diagnostics.

---

# 79. Cancellation Race Example

Scenario:

```text
Appointment A at 10:00

Request 1:
Cancel A

Request 2:
Book new Appointment at 10:00
```

With Staff lock:

```text
one transaction acquires lock first
```

If cancel first:

```text
cancel commits
booking rechecks
slot free
booking succeeds
```

If booking first:

```text
booking rechecks
A still blocks
booking fails conflict
cancel then commits
```

Both results are consistent.

---

# 80. Reschedule Race Example

Scenario:

```text
Appointment A at 10:00
Appointment B at 11:00

User 1:
Move A → 11:00

User 2:
Move B → 10:00
```

Both use the same Staff lock.

Operations serialize.

Each rechecks the current state after lock acquisition.

No invalid overlapping final state may commit.

---

# 81. Different Staff Concurrency

Bookings for:

```text
Staff A
Staff B
```

do not share a lock.

They proceed concurrently.

This preserves useful parallelism.

---

# 82. Same Staff Different Dates

Bookings for the same Staff on different dates serialize briefly under MVP.

This is accepted.

The transaction must be short enough that the practical impact remains negligible at expected scale.

---

# 83. Integration Test — Same Slot

Required test:

```text
Two concurrent requests
Same Tenant
Same Staff
Same interval

Expected:
exactly one succeeds
exactly one receives conflict
```

The test must run concurrently enough to exercise the race rather than sequentially simulate it.

---

# 84. Integration Test — Partial Overlap

Example:

```text
Request A: 10:00–11:00
Request B: 10:30–11:30
```

Expected:

```text
only one can reserve
```

---

# 85. Integration Test — Containment

Example:

```text
Existing: 10:00–12:00
Requested: 10:30–11:00
```

Expected:

```text
conflict
```

Reverse:

```text
Existing: 10:30–11:00
Requested: 10:00–12:00
```

Expected:

```text
conflict
```

---

# 86. Integration Test — Adjacent

Example:

```text
Existing: 10:00–11:00
Requested: 11:00–12:00
```

Expected:

```text
allowed
```

This validates half-open interval semantics.

---

# 87. Integration Test — Different Staff

Example:

```text
Staff A: 10:00–11:00
Staff B: 10:00–11:00
```

Expected:

```text
both allowed
```

unless another shared resource is introduced later.

---

# 88. Integration Test — Cancelled Appointment

Example:

```text
Cancelled Appointment: 10:00–11:00
Requested: 10:00–11:00
```

Expected:

```text
allowed
```

assuming Cancelled is non-blocking.

---

# 89. Integration Test — Tenant Isolation

Same Staff GUID collision should be impossible under normal keys, but Tenant scope must still be validated.

Test:

```text
Tenant A cannot lock/use Tenant B Staff
```

and:

```text
Tenant A Appointment query
cannot consider Tenant B data
```

---

# 90. Integration Test — Reschedule

Concurrent create/reschedule against the same Staff and interval must produce a consistent single reservation outcome.

---

# 91. Integration Test — Staff Schedule Change

Where schedule-change implementation is available:

```text
Concurrent booking
+
schedule update
```

must serialize through the same Staff booking guard.

Final state must satisfy the selected schedule-change business policy.

---

# 92. Integration Test Database

Concurrency tests must run against the real relational provider used by the environment under test.

Do not rely solely on:

```text
EF Core InMemory provider
```

for locking/concurrency behavior.

Provider concurrency semantics are part of what is being tested.

---

# 93. SQLite

SQLite may be useful for some tests, but it must not be treated as proof that SQL Server/PostgreSQL locking behavior works identically.

Critical appointment concurrency integration tests should target the production-equivalent database provider.

---

# 94. Testcontainers

Where practical, integration tests may use a real database container to exercise provider locking behavior consistently in CI.

Exact infrastructure choice belongs to test implementation.

---

# 95. Deadlock Handling

Even with deterministic lock ordering, database deadlocks may still occur in broader transactions.

Deadlock errors should be treated as transient technical failures where retry is safe.

The retry must rerun the complete transactional booking operation.

Deadlocks must be observable.

---

# 96. CancellationToken

All lock/transaction database operations must respect request cancellation where technically supported.

Abandoned requests should not hold locks longer than necessary.

---

# 97. Command Timeout

Booking writes should have reasonable database command timeouts.

Do not use extremely long timeouts to hide lock contention.

Persistent lock timeout indicates an operational/design problem that should be measured.

---

# 98. Unit Tests

Pure overlap logic should have unit/domain tests.

Test matrix should cover:

```text
same interval
partial overlap left
partial overlap right
contained interval
containing interval
adjacent before
adjacent after
separate intervals
```

Concurrency locking itself requires integration tests.

---

# 99. Domain Helper

Interval overlap logic should exist in one reliable domain/application abstraction.

Avoid repeating custom predicates in:

```text
create
reschedule
public booking
calendar
available slots
```

Read and write paths should agree on interval semantics.

---

# 100. Availability Engine Reuse

Internal appointment creation and public booking must use the same core:

```text
schedule resolution
interval rules
blocking-status definition
overlap semantics
```

The authoritative write path additionally applies the Staff booking lock.

---

# 101. Availability Query vs Booking Command

Architecture distinction:

```text
Availability Query
→ advisory snapshot
→ no long-lived lock

Booking Command
→ authoritative
→ transaction + lock + recheck
```

This separation is intentional.

---

# 102. Reschedule Exclusion

When checking overlap during reschedule:

```text
exclude current Appointment Id
```

Otherwise the Appointment conflicts with itself.

The exclusion must remain Tenant-scoped and Staff-scoped.

---

# 103. Appointment State Changes

State transition and concurrency validation happen together.

Example cancellation:

```text
lock
↓
load current version
↓
validate current state allows Cancel
↓
change state
↓
save
```

Do not validate state only before acquiring relevant concurrency protection and then assume it remains unchanged.

---

# 104. Lost Update Protection

Where a client edits Appointment details, require a concurrency/version value if the chosen API contract supports it.

Conceptual:

```text
AppointmentId
Version
Changes
```

If stale:

```text
409 appointment.concurrent_update
```

The exact DTO strategy may evolve.

---

# 105. Audit of Reschedule

Reschedule audit should record enough information to understand the change:

```text
Old Staff
Old Start/End
Old Branch

New Staff
New Start/End
New Branch

Actor
Timestamp
```

Audit should be committed consistently with the Appointment change or reliably emitted after commit.

---

# 106. Payment Does Not Hold Booking Lock

Payment processing should not hold a Staff booking lock unless it changes Appointment reservation state.

Example:

```text
Appointment already created
→ register payment
```

does not need Staff schedule serialization.

Keep lock scope focused.

---

# 107. Force Appointment Future

Force Appointment intentionally allows an exceptional booking workflow.

It must not bypass concurrency safety.

A future Force Appointment may bypass:

```text
normal schedule capacity rule
```

only under explicit business policy.

It must still serialize against the Staff booking resource so two Force requests cannot accidentally create an invalid state.

Force rules belong to a later feature/ADR if needed.

---

# 108. Manual Override Future

If Owner/Admin receives an override capability:

```text
Override schedule
```

the system must distinguish:

```text
explicit authorized override
```

from:

```text
race-condition-created overlap
```

Concurrency safety remains mandatory.

Any intentional overlapping Appointment capability requires explicit product semantics.

---

# 109. Multi-Staff Appointment Future

If a Service later requires multiple Staff members:

```text
Acquire all required Staff locks
in deterministic order
↓
validate all availability
↓
commit atomically
```

This ADR's multi-lock ordering rule already supports that extension.

---

# 110. Database Provider Change

If Shinera changes database provider:

The new provider must implement and verify:

```text
Staff booking lock
transaction lifecycle
timeout behavior
deadlock/retry behavior
```

Appointment concurrency tests are mandatory before migration is considered safe.

---

# 111. Implementation Boundary

Application layer should express intent:

```text
Acquire booking protection for Staff
```

Infrastructure should express database mechanics:

```text
FOR UPDATE
UPDLOCK
provider transaction behavior
```

Do not leak raw provider locking syntax throughout handlers.

---

# 112. Example Abstraction

Conceptual only:

```csharp
public interface IAppointmentConcurrencyGuard
{
    Task AcquireStaffLockAsync(
        Guid tenantId,
        Guid staffId,
        CancellationToken cancellationToken);
}
```

For multi-lock operations, implementation may accept an ordered collection.

Exact API should follow actual codebase style.

---

# 113. Transaction Ownership

The application service/handler orchestrating Appointment mutation owns the transaction boundary, or calls a dedicated application abstraction that does.

The lock acquisition must occur inside the active transaction.

A lock acquired outside the transaction and released before `SaveChanges()` is useless.

---

# 114. Guard Requirement

`AcquireStaffLockAsync` must fail if:

```text
Staff does not exist
or
Staff does not belong to Current Tenant
```

It must never silently create access to a foreign Staff row.

---

# 115. Lock Reentrancy

Within one transaction, multiple validations may reference the same Staff.

The concurrency abstraction should avoid unsafe duplicate lock behavior but does not require a complex reentrant lock framework.

Database semantics naturally allow the transaction to continue operating on the row it already locked.

---

# 116. Error Exposure

Do not expose raw database messages such as:

```text
deadlock victim
could not serialize access
lock request timeout
```

to end users.

Map them to stable application behavior and log technical details with TraceId.

---

# 117. User Experience

When `appointment.conflict` occurs:

Frontend should:

```text
tell user that the slot was just taken
refresh availability
preserve selected Customer/Service/Staff where possible
ask for another time
```

Do not show a generic "500" error.

---

# 118. Public Booking UX

For a public customer, the conflict message should remain simple:

```text
This time is no longer available.
Please choose another time.
```

The customer does not need database concurrency details.

---

# 119. Security

Concurrency error behavior must not reveal cross-tenant Appointment information.

Do not say:

```text
Appointment #123 in Salon B occupies this slot
```

Only return the current Tenant's business-safe conflict result.

---

# 120. Definition of Done Integration

Any Story that:

```text
creates
reschedules
cancels
or materially changes Staff bookability
```

must explicitly evaluate Appointment concurrency.

For Appointment creation/reschedule:

```text
Concurrency = mandatory DoD gate
```

Required evidence includes:

```text
transaction exists
Staff lock acquired
final overlap recheck occurs after lock
conflict mapped correctly
concurrent integration test passes
```

---

# 121. Rejected Alternatives Summary

Rejected as primary strategy:

```text
Frontend-only slot disabling
Check-then-insert without lock
C# process lock
SemaphoreSlim
Tenant-level lock
Branch-level lock
Appointment table lock
Optimistic concurrency alone
Unique StartTime constraint alone
Provider-specific exclusion constraint as sole guarantee
Global SERIALIZABLE for all application transactions
Redis/distributed lock as unnecessary extra authority
```

---

# 122. Consequences

## Positive

This decision provides:

```text
Strong double-booking protection
Multi-instance safety
Short transaction boundaries
Clear serialization resource
Database-provider flexibility
Simple mental model
Easy code review
Good future extensibility
```

## Negative

It introduces:

```text
Provider-specific infrastructure lock implementations
Brief serialization of same-Staff bookings across dates
Need for real database concurrency tests
Potential lock waits
Need for retry/timeout discipline
```

These costs are accepted.

---

# 123. Revisit Conditions

Revisit this ADR if:

```text
booking lock contention becomes measurable
multi-resource booking is introduced
cross-midnight scheduling becomes required
database provider becomes permanently fixed and offers better safe constraints
multiple Staff per Appointment becomes common
room/equipment capacity becomes core
intentional overlapping/force capacity becomes a major product feature
```

Until then, avoid increasing concurrency complexity.

---

# 124. Final Decision Summary

Shinera prevents Appointment double booking using:

```text
Short DB Transaction
+
Pessimistic Staff Booking Lock
+
Final Availability Revalidation
+
Correct Interval Overlap Query
+
Atomic Appointment Write
```

The lock key is initially:

```text
Tenant + Staff
```

Appointment intervals use:

```text
[Start, End)
```

with overlap:

```text
existing.Start < requested.End
AND
requested.Start < existing.End
```

Availability queries are advisory.

Create/reschedule commands are authoritative.

Optimistic concurrency tokens protect edits to an existing Appointment but do not replace Staff booking serialization.

All application instances coordinate through the relational database, not process-local locks.

This strategy prioritizes:

```text
Correctness
Security
Auditability
Simplicity
```

over premature maximum concurrency.

---

# 125. Official Technical References

- EF Core concurrency handling:
  https://learn.microsoft.com/ef/core/saving/concurrency

- SQL Server transaction locking and row versioning:
  https://learn.microsoft.com/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide

- SQL Server transaction isolation:
  https://learn.microsoft.com/sql/t-sql/statements/set-transaction-isolation-level-transact-sql

- PostgreSQL transaction isolation:
  https://www.postgresql.org/docs/current/transaction-iso.html

- PostgreSQL explicit locking:
  https://www.postgresql.org/docs/current/explicit-locking.html
