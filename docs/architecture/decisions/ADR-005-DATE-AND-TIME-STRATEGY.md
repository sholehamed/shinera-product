# ADR-005 — Date and Time Strategy

**Project:** Shinera  
**Status:** Accepted  
**Date:** 2026-10-05  
**Decision Type:** Architecture / Temporal Modeling / Localization  
**Scope:** Backend, Frontend, Database, Appointments, Schedules, Reports, Audit, Testing  
**Related Documents:**
- `SHINERA-PRD.md`
- `SHINERA-PRODUCT-BACKLOG.md`
- `PROJECT-INSTRUCTIONS.md`
- `ARCHITECTURE-OVERVIEW.md`
- `DEFINITION-OF-DONE.md`
- `ADR-001-MULTI-TENANCY-STRATEGY.md`
- `ADR-004-APPOINTMENT-CONCURRENCY-STRATEGY.md`

---

# 1. Context

Shinera is a Persian-first scheduling SaaS.

Date and time appear in nearly every important domain:

```text
Staff schedules
Appointments
Breaks
Days off
Time off
Payments
Subscription
Audit
Dashboard
Reports
Notifications
Created/Updated timestamps
```

Several different temporal concepts exist and must not be modeled as if they were the same thing.

Examples:

```text
"Staff works every Saturday at 09:00"
```

is a local wall-clock schedule rule.

```text
"Appointment starts at this exact moment"
```

is a globally unambiguous instant.

```text
"1405/07/13"
```

is a Persian-calendar presentation of a civil date.

```text
"CreatedAt"
```

is an audit timestamp.

Using one type and one storage rule for all of these creates subtle bugs.

---

# 2. Decision

Shinera will separate temporal concepts into three categories:

```text
1. Instant
   → exact point on the global timeline
   → stored in UTC

2. Business Local Date/Time
   → wall-clock values meaningful inside a Branch
   → DateOnly / TimeOnly

3. Presentation Calendar
   → Persian/Jalali or localized formatting
   → UI concern only
```

The canonical architecture is:

```text
Instant
→ DateTimeOffset normalized to UTC

Business Date
→ DateOnly

Business Time
→ TimeOnly

Duration
→ TimeSpan or explicit minute value depending on domain contract

Time Zone
→ IANA time zone identifier

Persian Calendar
→ presentation only
```

---

# 3. Fundamental Rule

Never ask:

```text
"What type do we use for dates?"
```

Ask:

```text
"What kind of time concept is this?"
```

The answer determines the type and persistence behavior.

---

# 4. Temporal Type Matrix

| Concept | .NET Representation | Canonical Meaning |
|---|---|---|
| CreatedAt / UpdatedAt | `DateTimeOffset` UTC | Global instant |
| PaidAt | `DateTimeOffset` UTC | Global instant |
| Token/session timestamps | UTC instant | Global instant |
| Appointment Start/End | `DateTimeOffset` UTC | Global instant |
| Appointment BusinessDate | `DateOnly` | Branch-local business date |
| Weekly schedule start/end | `TimeOnly` | Branch-local wall time |
| Day off | `DateOnly` | Branch-local business date |
| Special schedule date | `DateOnly` | Branch-local business date |
| Time-off date/time | local date/time resolved through Branch TZ | Business wall time |
| Service duration | duration value | Elapsed amount |
| UI Persian date | presentation | Not persisted as canonical value |

---

# 5. UTC Instant Representation

Exact instants are represented in application code as:

```csharp
DateTimeOffset
```

normalized to:

```text
Offset = +00:00
```

Examples:

```text
CreatedAtUtc
UpdatedAtUtc
StartUtc
EndUtc
PaidAtUtc
CancelledAtUtc
CompletedAtUtc
```

Naming should make UTC semantics obvious where useful.

---

# 6. Why DateTimeOffset

`DateTimeOffset` represents an unambiguous point in time because it carries an offset from UTC.

Shinera standardizes persisted instant values to:

```text
UTC / offset zero
```

to avoid mixed-offset data.

A `DateTimeOffset` value still does **not** identify a time zone.

Therefore:

```text
Instant
+
TimeZoneId
```

are separate concepts.

---

# 7. Do Not Store Arbitrary Offsets

Persisted global instants should not contain arbitrary local offsets such as:

```text
+03:30
+04:00
```

as the canonical database value.

Normalize:

```text
2026-10-05T12:00:00+03:30
```

to:

```text
2026-10-05T08:30:00Z
```

before persistence as a canonical instant.

---

# 8. Time Zone Identifier

Shinera uses:

```text
IANA time zone identifiers
```

as the canonical application-level identifier.

Examples:

```text
Asia/Tehran
Asia/Dubai
Europe/Istanbul
America/Toronto
```

Avoid storing raw offsets as the business time zone.

Bad:

```text
+03:30
```

because an offset does not contain time-zone rules.

---

# 9. Why IANA

IANA identifiers are:

- standard across Linux/cloud environments,
- rule-aware,
- suitable for daylight-saving transitions,
- more expressive than a fixed offset.

Shinera backend should resolve them through a centralized timezone abstraction using .NET time-zone APIs.

Runtime environments must include appropriate globalization/time-zone data.

---

# 10. Tenant Default Time Zone

Tenant contains a default time zone.

Conceptually:

```text
Tenant.DefaultTimeZoneId
```

Example:

```text
Asia/Tehran
```

This value is chosen during:

```text
registration
or
workspace configuration
```

and becomes the default for newly created Branches.

---

# 11. Branch Time Zone

Each Branch has its own required time zone:

```text
Branch.TimeZoneId
```

When a Branch is created:

```text
Branch.TimeZoneId
=
Tenant.DefaultTimeZoneId
```

unless explicitly configured otherwise.

This allows:

```text
Tenant
├── Tehran Branch → Asia/Tehran
└── Dubai Branch  → Asia/Dubai
```

without redesigning scheduling.

---

# 12. Why Time Zone Belongs to Branch

Appointments and schedules happen at a physical/operational location.

Therefore the Branch is the natural business-time boundary.

A Tenant-wide timezone alone would prevent correct future multi-region operation.

---

# 13. Solo Businesses

Solo mode still uses:

```text
Tenant
+
Main Branch
```

Therefore Solo scheduling uses:

```text
MainBranch.TimeZoneId
```

with no separate temporal model.

---

# 14. Time Zone Change

Changing:

```text
Branch.TimeZoneId
```

affects future business-time resolution.

It must not silently rewrite historical appointment meaning.

Historical Appointments preserve enough temporal snapshot data to remain interpretable.

---

# 15. Appointment Temporal Model

An Appointment should persist at minimum:

```text
StartUtc
EndUtc
BusinessDate
TimeZoneId
```

where:

```text
StartUtc
→ exact start instant

EndUtc
→ exact end instant

BusinessDate
→ local Branch date on which the booking belongs

TimeZoneId
→ time zone used when booking was resolved
```

Exact property naming may vary.

---

# 16. Why Appointment Stores BusinessDate

`BusinessDate` is intentionally persisted even though it can often be derived from:

```text
StartUtc + TimeZoneId
```

Reasons:

- efficient "appointments for this business day" queries,
- stable reporting semantics,
- historical consistency after Branch timezone changes,
- clear schedule partitioning,
- simple concurrency/index strategy.

It represents the business's local civil date at booking time.

---

# 17. Appointment TimeZoneId Snapshot

Appointment stores the time-zone identifier used when the booking interval was created.

This protects history from later Branch configuration changes.

Example:

```text
Appointment booked under Asia/Tehran
```

remains historically associated with that zone even if the Branch is later moved/configured differently.

---

# 18. Appointment Local Time

Canonical scheduling input is:

```text
BusinessDate
+
Local Start Time
+
Branch TimeZone
```

Backend resolves this into:

```text
StartUtc
```

Then:

```text
EndUtc = StartUtc + effective duration
```

subject to schedule/DST validation.

The final Appointment persists the resulting UTC interval.

---

# 19. Appointment Request Contract

Preferred API contract:

```json
{
  "branchId": "...",
  "staffId": "...",
  "serviceId": "...",
  "customerId": "...",
  "date": "2026-10-05",
  "startTime": "09:30:00"
}
```

The client does not need to send a trusted timezone offset.

Backend resolves:

```text
BranchId
→ Branch.TimeZoneId
→ local date/time
→ UTC instant
```

---

# 20. API Local Date Format

Canonical local date format:

```text
YYYY-MM-DD
```

Example:

```text
2026-10-05
```

This is Gregorian ISO-style representation.

Do not send:

```text
1405/07/13
```

as the canonical API date.

---

# 21. API Local Time Format

Canonical local time:

```text
HH:mm:ss
```

or compatible ISO local-time representation.

Example:

```text
09:30:00
```

Seconds may be omitted by UI when not required, but backend contracts should remain unambiguous.

---

# 22. API Instant Format

Canonical instant output:

```text
ISO 8601 UTC
```

Example:

```text
2026-10-05T06:00:00Z
```

Do not emit server-machine local time as API canonical timestamp.

---

# 23. Persian Calendar

The Persian/Jalali calendar is a display/input experience.

Architecture rule:

```text
User sees Persian date
↓
Frontend converts to canonical Gregorian DateOnly
↓
API sends YYYY-MM-DD
↓
Backend stores DateOnly / UTC values
```

Reverse:

```text
Backend returns canonical date/time
↓
Frontend formats as fa-IR / Persian calendar
```

---

# 24. Why Jalali Is Not Canonical Storage

Do not persist values such as:

```text
"1405/07/13"
```

as the canonical business date.

Reasons:

- string parsing ambiguity,
- harder indexing,
- provider incompatibility,
- unnecessary calendar coupling,
- conversion duplicated across layers.

Calendar is representation.

Civil business date remains canonical Gregorian `DateOnly`.

---

# 25. Localized Digits

API contracts use ASCII digits.

Example:

```text
2026-10-05
```

not:

```text
۲۰۲۶-۱۰-۰۵
```

Persian digits belong to presentation.

---

# 26. Weekly Staff Schedule

Recurring weekly schedules use local wall-clock values.

Conceptually:

```text
StaffWeeklySchedule
├── DayOfWeek
├── StartTime : TimeOnly
└── EndTime   : TimeOnly
```

Example:

```text
Saturday
09:00
18:00
```

These are interpreted in:

```text
Branch.TimeZoneId
```

for the requested business date.

---

# 27. Why Schedule Is Not Stored in UTC

A rule such as:

```text
Every Saturday 09:00–18:00
```

means wall-clock time at the Branch.

If stored only as UTC, daylight-saving or timezone rule changes could shift the visible work hours.

Recurring schedules therefore remain:

```text
local business time
+
time zone context
```

not fixed UTC instants.

---

# 28. Breaks

Recurring breaks use:

```text
Day/relationship to schedule
+
TimeOnly Start
+
TimeOnly End
```

Example:

```text
13:00–14:00
```

They inherit the Staff/Branch timezone context.

---

# 29. Day Off

A full day off is represented as:

```text
DateOnly
```

in the relevant Branch business calendar/date model.

It is not:

```text
00:00 UTC → 24:00 UTC
```

because a local business day may not align with UTC boundaries.

---

# 30. Special Schedule

A special schedule uses:

```text
DateOnly
+
TimeOnly Start
+
TimeOnly End
```

interpreted through the Branch timezone.

It overrides or supplements recurring weekly rules according to the Schedule domain policy.

---

# 31. Time Off

Time off may require either:

```text
full-day DateOnly range
```

or:

```text
local date + local time interval
```

depending on product requirements.

The domain should model the actual business concept rather than forcing every time-off record into UTC-only form.

When evaluating against Appointments, local schedule values are resolved to UTC intervals.

---

# 32. Duration

Duration is not a time of day.

Use:

```text
TimeSpan
```

inside domain logic where appropriate.

For simple API/database contracts, explicit integer minutes may be clearer:

```text
DurationMinutes
```

Example:

```text
Service.DurationMinutes = 45
```

Do not use `TimeOnly` for duration.

---

# 33. DateOnly / TimeOnly

Shinera uses native .NET:

```text
DateOnly
TimeOnly
```

for business local dates and times.

Do not use:

```text
DateTime at midnight
```

to represent a date-only value.

Do not use:

```text
TimeSpan
```

to represent a clock time when `TimeOnly` is semantically correct.

---

# 34. Database Mapping

Provider-specific persistence should preserve semantic intent.

Conceptually:

```text
DateOnly
→ date

TimeOnly
→ time

DateTimeOffset UTC instant
→ provider-appropriate timestamp/offset type
```

For PostgreSQL, point-in-time values must be persisted with UTC semantics.

For SQL Server, UTC `DateTimeOffset` values may map to an appropriate `datetimeoffset` representation.

Infrastructure owns provider-specific mapping details.

---

# 35. PostgreSQL Rule

If PostgreSQL is used:

```text
timestamp with time zone / timestamptz
```

represents an instant and does not preserve the original IANA timezone identifier.

Therefore Shinera still stores:

```text
TimeZoneId
```

separately when timezone identity matters.

UTC values are required for canonical instant persistence.

---

# 36. Database Server Time Zone

Application correctness must not depend on:

```text
database server local timezone
```

or:

```text
application server local timezone
```

Servers should preferably run UTC.

Business timezone always comes from explicit Shinera configuration.

---

# 37. Do Not Use Server Local Time

Avoid business logic based on:

```csharp
DateTime.Now
```

or:

```csharp
DateTime.Today
```

because these use the host machine's local timezone.

The server's physical location is irrelevant to the salon's business time.

---

# 38. Current Time Abstraction

Shinera uses:

```text
TimeProvider
```

for current-time access.

Preferred:

```csharp
timeProvider.GetUtcNow()
```

Do not scatter:

```csharp
DateTime.UtcNow
DateTimeOffset.UtcNow
```

through application/domain logic.

---

# 39. Why TimeProvider

A centralized time source provides:

- deterministic tests,
- controllable current time,
- cleaner domain orchestration,
- easier boundary testing.

Application code obtains "now" from `TimeProvider`.

Domain entities may receive the instant as a method argument rather than directly reading system time.

---

# 40. Example Domain Call

Preferred conceptual flow:

```text
Application Handler
↓
now = TimeProvider.GetUtcNow()
↓
appointment.Cancel(now)
```

rather than Domain calling:

```text
DateTime.UtcNow
```

internally.

This keeps Domain behavior deterministic.

---

# 41. Business "Today"

There is no universal Shinera business `Today`.

Business today means:

```text
Current Instant
↓
Convert using Branch.TimeZoneId
↓
DateOnly
```

Therefore:

```text
Tehran Branch Today
```

may differ from:

```text
Toronto Branch Today
```

at the same instant.

---

# 42. Tenant-Wide "Today"

If a Tenant has multiple Branches in different time zones, a single Tenant-wide "today" is ambiguous.

For Branch-filtered operational screens:

```text
use Branch business date
```

For Tenant-wide reports:

```text
use persisted BusinessDate per Branch
```

and make report semantics explicit.

Do not silently use API server timezone.

---

# 43. Dashboard Date Filters

Dashboard filters such as:

```text
Today
This Week
This Month
```

are business-local concepts.

For a selected Branch:

```text
resolve using Branch.TimeZoneId
```

For multi-Branch aggregate reporting:

```text
prefer BusinessDate-based aggregation
```

with an explicitly defined local reporting policy.

---

# 44. Appointment Query by Day

Preferred query semantics:

```text
TenantId
BranchId
BusinessDate
```

rather than computing:

```text
UTC day boundary
```

using server local time.

This supports efficient and correct Branch-local calendar views.

---

# 45. Appointment Concurrency Integration

ADR-004 uses:

```text
Tenant
+
Staff
+
BusinessDate/interval
```

semantics.

The final overlap comparison occurs on canonical resolved appointment intervals.

For exact overlap:

```text
StartUtc
EndUtc
```

are safe to compare globally.

BusinessDate remains useful for narrowing query scope when the domain does not support cross-midnight bookings.

---

# 46. Cross-Midnight Appointments

MVP default:

```text
Appointment must remain within one business date
```

unless a future product requirement explicitly supports overnight services.

This means a booking starting on one Branch-local date should not end on another Branch-local date.

This simplifies:

- schedule evaluation,
- day views,
- reporting,
- concurrency filtering.

Future overnight support requires ADR review.

---

# 47. Daylight-Saving Time

Even if the primary launch market does not currently use DST, architecture must not assume every future Branch timezone is fixed-offset.

Local wall time may be:

```text
valid
invalid
ambiguous
```

during timezone transitions.

---

# 48. Invalid Local Time

A local time can be nonexistent during a forward clock transition.

Example conceptually:

```text
02:30
```

may never occur on a particular date.

Backend must reject such booking input.

Suggested error:

```text
time.invalid_local_time
```

Do not silently shift the booking to another clock time.

---

# 49. Ambiguous Local Time

During a backward clock transition, a local clock time may occur twice.

Example:

```text
01:30
```

may represent two different UTC instants.

Default Shinera behavior:

```text
reject ambiguous local booking input
```

unless the client explicitly supplies a supported disambiguation mechanism.

Suggested error:

```text
time.ambiguous_local_time
```

Silent offset selection is rejected.

---

# 50. Why Reject Ambiguous Times

Automatically choosing:

```text
earlier offset
```

or:

```text
later offset
```

creates invisible booking semantics.

For a scheduling product, explicit failure is safer than guessing.

A future UI may present both occurrences if required.

---

# 51. Time Zone Resolution Service

Centralize timezone logic behind an abstraction.

Conceptual:

```text
ITimeZoneService
```

Responsibilities may include:

```text
Validate IANA zone
Resolve local date/time to UTC
Convert UTC instant to Branch local time
Detect invalid time
Detect ambiguous time
Get business date
```

Avoid scattered direct `TimeZoneInfo` conversion code.

---

# 52. Example Resolution

Conceptual:

```text
Date = 2026-10-05
Time = 09:30
Zone = Asia/Tehran
```

Backend:

```text
Local DateTime (Unspecified semantic)
↓
TimeZone resolution
↓
UTC instant
```

The intermediate value must not accidentally be interpreted as the server's local time.

---

# 53. DateTimeKind

When a `DateTime` is temporarily used to combine:

```text
DateOnly + TimeOnly
```

for `TimeZoneInfo` conversion, its semantic kind should be explicitly treated as:

```text
Unspecified
```

until the target time zone is applied.

Do not create a local wall time and mark it `Utc`.

---

# 54. Time Zone Validation

When saving:

```text
Tenant.DefaultTimeZoneId
Branch.TimeZoneId
```

backend must verify the identifier is recognized by the runtime timezone service.

Invalid timezone IDs are rejected.

---

# 55. Time Zone API

Frontend may obtain supported zones from:

```text
configuration/metadata endpoint
```

or a curated list.

Persisted value remains the canonical IANA ID.

Do not trust arbitrary UI labels as timezone identifiers.

---

# 56. Time Zone Display Name

Display names may be localized and can change.

Do not use human-readable display text as the persistent key.

Persist:

```text
Asia/Tehran
```

Display:

```text
تهران (UTC+03:30)
```

or equivalent UI text.

---

# 57. Default Time Zone

The product may suggest a timezone based on user/device/locale during registration.

But the final timezone is saved explicitly.

Do not silently make permanent business-time configuration depend on the backend server location.

---

# 58. Time Zone Changes and Future Appointments

Changing a Branch timezone may affect already scheduled future Appointments.

This is a business-sensitive operation.

MVP rule:

```text
Timezone change must not automatically reinterpret existing Appointment UTC instants.
```

Existing appointments remain fixed instants with their historical timezone snapshot.

Future availability uses the new Branch timezone.

The UI should warn if future appointments exist before timezone change.

A more advanced migration/reinterpretation workflow may be designed later.

---

# 59. Schedule Interpretation After Time Zone Change

Recurring Staff schedules are wall-clock rules.

After Branch timezone changes:

```text
09:00 remains 09:00 local
```

under the new timezone for future dates.

This is intentional because schedules describe local working hours.

Existing Appointment UTC instants are not automatically shifted.

---

# 60. Audit Timestamps

Audit records use:

```text
UTC instant
```

Example:

```text
OccurredAtUtc
```

Display converts to the viewer's chosen/business timezone.

Audit ordering must not depend on localized strings.

---

# 61. CreatedAt / UpdatedAt

Entity audit timestamps use:

```text
UTC
```

and are populated consistently.

Prefer centralized audit handling where already established.

These fields represent technical/business event instants, not Branch wall-clock schedule values.

---

# 62. Payment Time

Payment event time uses:

```text
PaidAtUtc
```

If financial/reporting grouping by business date is needed, compute or persist the relevant business date explicitly.

Do not infer it from server timezone.

---

# 63. Subscription Time

Subscription boundaries represent exact platform instants unless the product explicitly defines them as business dates.

Examples:

```text
StartedAtUtc
ExpiresAtUtc
```

If a plan expires "at end of local business day", that rule must explicitly resolve the Tenant/Branch timezone.

Do not mix date-only and instant semantics.

---

# 64. Notification Scheduling

Appointment reminder:

```text
Appointment.StartUtc
-
ReminderLeadTime
```

produces a UTC execution instant.

Background workers operate on UTC instants.

Notification display may include Branch-local formatted time.

---

# 65. Background Jobs

Background jobs must never assume process local timezone.

Use:

```text
UTC instant
+
explicit TimeZoneId
```

for tenant-aware scheduling.

Recurring business-time jobs need explicit timezone-aware recurrence logic.

---

# 66. Serialization

JSON should use standard ISO representations.

Examples:

```json
{
  "date": "2026-10-05",
  "startTime": "09:30:00",
  "createdAtUtc": "2026-10-05T06:02:45Z"
}
```

Do not use culture-specific strings in machine contracts.

---

# 67. Frontend Date Model

Angular should distinguish:

```text
canonical API date
display Persian date
instant timestamp
```

Do not pass formatted Persian strings directly into API models.

Create conversion/adaptation at UI boundaries.

---

# 68. JavaScript Date Warning

JavaScript `Date` represents an instant-like timestamp and automatically applies environment timezone behavior.

Do not use it blindly for:

```text
DateOnly
TimeOnly
Persian calendar local date
```

A selected business date such as:

```text
2026-10-05
```

should not accidentally become the previous/next day due to browser timezone conversion.

---

# 69. Frontend DateOnly Handling

For date-only API values:

```text
"2026-10-05"
```

treat them as civil date values.

Do not automatically parse them into UTC midnight and then display them through local timezone conversion.

That can shift the visible date.

---

# 70. Frontend TimeOnly Handling

For:

```text
"09:30:00"
```

treat it as a wall-clock time.

Do not attach the browser's timezone unless intentionally resolving an Appointment instant.

---

# 71. Browser Time Zone

The customer's browser timezone is not automatically the salon Branch timezone.

Public booking should display appointment times in the business timezone by default.

If future UX offers customer-local conversion, it must clearly label the timezone.

---

# 72. Public Booking

Public booking resolves:

```text
Tenant
↓
Branch
↓
Branch.TimeZoneId
↓
Business date/time
```

The backend remains authoritative for conversion and availability.

Browser-provided offset is advisory at most.

---

# 73. Persian Week Start

UI calendar may present the Persian week beginning on:

```text
Saturday
```

but backend schedule logic may use .NET's standard `DayOfWeek` representation.

Presentation ordering and domain enum numeric representation are separate.

Do not depend on enum numeric value for UI ordering.

---

# 74. DayOfWeek Persistence

If DayOfWeek is persisted as an integer or enum:

```text
meaning must be stable and documented
```

UI should map it explicitly.

Do not assume:

```text
0 = first day shown in Persian calendar
```

---

# 75. Month Boundaries

Business reporting month may mean:

```text
Gregorian month
```

or:

```text
Persian calendar month
```

These are different product concepts.

MVP dashboard/reporting must explicitly define which one is intended.

If the UI offers Persian-month reporting, backend should receive resolved Gregorian date boundaries or a clearly defined reporting period contract.

Do not silently infer based on UI language.

---

# 76. Recommended MVP Reporting Rule

Operational appointment lists use:

```text
BusinessDate
```

For dashboard shortcuts:

```text
Today / This Week
```

use Branch-local civil dates.

For:

```text
This Month
```

if Persian calendar semantics are displayed to Persian users, the frontend or shared calendar service must resolve the Persian month to an explicit Gregorian date range.

The API should receive unambiguous date boundaries.

---

# 77. Query Range Contract

Preferred report/list query:

```text
fromDate: YYYY-MM-DD
toDate: YYYY-MM-DD
branchId
```

with documented inclusive/exclusive semantics.

Recommended:

```text
fromDate inclusive
toDate exclusive
```

for range composition.

For simple single-day filters:

```text
date = YYYY-MM-DD
```

is clearer.

---

# 78. Instant Range Contract

For technical/audit instant filters:

```text
fromUtc inclusive
toUtc exclusive
```

Use:

```text
[start, end)
```

consistently.

This aligns with Appointment interval semantics in ADR-004.

---

# 79. Precision

Shinera does not require nanosecond-level business precision.

Business appointment UI normally operates at:

```text
minute precision
```

but persisted UTC timestamp precision may be higher.

Avoid equality comparisons on current timestamps when range comparison is intended.

---

# 80. Appointment Slot Precision

Availability slot granularity may be configured separately from timestamp storage precision.

Example:

```text
15-minute slot step
```

does not mean the database timestamp type should only support 15-minute precision.

Slot generation is a business rule.

---

# 81. Service Duration Precision

MVP service duration should normally use whole minutes.

Example:

```text
30
45
60
90
```

If seconds-level duration becomes necessary later, the contract may evolve.

Do not overcomplicate service duration initially.

---

# 82. Testing Current Time

Tests must not depend on the real wall clock for business rules.

Use:

```text
FakeTimeProvider
```

or equivalent controlled `TimeProvider`.

Examples:

```text
subscription expiration
same-day behavior
appointment cancellation cutoff
dashboard today
reminder timing
```

---

# 83. Temporal Test Matrix

Important tests include:

```text
UTC conversion
business date conversion
timezone validation
midnight boundary
different Branch timezones
invalid DST local time
ambiguous DST local time
adjacent Appointment intervals
month/week boundaries
timezone change behavior
```

---

# 84. Midnight Boundary Test

Example:

```text
Instant: 20:45 UTC

Branch A local date: October 5
Branch B local date: October 6
```

Business-date functions must return Branch-specific results.

Do not test only one timezone.

---

# 85. Persian Conversion Tests

Frontend/shared calendar conversion should verify round-trip behavior:

```text
Persian user-selected date
→ canonical Gregorian DateOnly
→ Persian display
```

using known boundary dates.

Backend does not need to know Jalali strings for ordinary appointment contracts.

---

# 86. DST Tests

Even if launch timezone has no active DST transition, include timezone test data from a zone that does.

This ensures the generic time-zone resolver correctly handles:

```text
invalid
ambiguous
normal
```

local times.

---

# 87. Database Integration Tests

Provider integration tests should verify:

```text
UTC instant round-trip
DateOnly round-trip
TimeOnly round-trip
precision behavior
query range behavior
```

against the actual relational provider.

---

# 88. Migration Review

Temporal column migrations require careful review.

Common dangerous changes:

```text
timestamp without timezone
→ timestamp with timezone

DateTime
→ DateOnly

local stored time
→ UTC
```

Existing data cannot be safely converted without knowing its original semantics.

Do not allow an automatic migration to guess old timezone meaning.

---

# 89. Legacy Data Conversion

If old data contains local `DateTime` without a known timezone:

```text
stop
identify original timezone semantics
write explicit migration strategy
```

Do not call:

```text
ToUniversalTime()
```

on unknown historical values and assume correctness.

---

# 90. Database Defaults

Avoid database defaults based on local server time such as:

```text
GETDATE()
local timestamp
```

for canonical UTC audit fields.

If database-generated timestamps are used, use provider-appropriate UTC expressions.

Application-level `TimeProvider` may be preferred for consistent domain timestamps.

---

# 91. Time Comparison

Compare:

```text
instants with instants
local dates with local dates
local times with local times
durations with durations
```

Avoid implicit mixing.

Example bad comparison:

```text
Appointment.StartUtc.Date
==
Branch local DateOnly
```

without timezone conversion.

---

# 92. Expiration Checks

Expiration:

```text
ExpiresAtUtc <= nowUtc
```

uses UTC instants.

Do not convert to Persian/local date unless the business rule explicitly says expiration occurs at local-day boundary.

---

# 93. Business-Day Boundary Rules

If a future business defines:

```text
business day ends at 02:00
```

instead of midnight, that is a separate business-calendar concept.

Current MVP assumes:

```text
business date boundary = local midnight
```

unless explicitly changed.

---

# 94. Holidays

Holiday calendars are not part of the core temporal architecture.

A future holiday feature should use:

```text
DateOnly
+
business/region context
```

not global UTC timestamps.

---

# 95. Recurrence

Recurring schedules are modeled as business recurrence rules, not pre-generated UTC appointments.

Example:

```text
Every Saturday 09:00
```

should be resolved per requested date using the Branch timezone.

This keeps recurrence correct across timezone rule changes.

---

# 96. Time Zone Data Updates

IANA timezone rules may change over time.

Runtime/container images must receive normal operating-system/globalization updates.

Shinera should not hardcode DST transition tables.

Historical Appointment `TimeZoneId` and `BusinessDate` snapshots reduce dependence on current Branch settings.

---

# 97. Container Requirements

Linux production containers must include the runtime data required for:

```text
IANA time zones
globalization
Persian/localized formatting where backend requires it
```

Do not enable globalization-invariant deployment if timezone/localization features require ICU behavior that would be lost.

Deployment tests should validate required timezone IDs.

---

# 98. Time Zone Conversion Portability

If Shinera executes on Windows environments, timezone resolution should use a centralized adapter capable of handling the canonical IANA identifier strategy with supported .NET/ICU conversion facilities where needed.

Do not spread OS-specific timezone IDs through domain data.

---

# 99. Error Codes

Suggested temporal errors:

```text
time.invalid_timezone
time.invalid_local_time
time.ambiguous_local_time
time.invalid_range
appointment.cross_day_not_supported
```

Frontend should react to stable codes rather than parse text.

---

# 100. Validation Rules

Common validation:

```text
End > Start
Valid TimeZoneId
Date within supported booking range
Local time exists in timezone
Local time is unambiguous
Appointment stays in one business date for MVP
```

Schedule-specific validation may additionally prevent invalid break/work intervals.

---

# 101. Authorization and Time

Permission decisions must not depend on browser-local dates.

If a permission/business rule has a temporal condition:

```text
resolve using authoritative UTC/business timezone
```

server-side.

---

# 102. Cancellation Cutoff Example

Future rule:

```text
Customer may cancel until 2 hours before Appointment
```

Correct:

```text
Appointment.StartUtc - nowUtc >= 2h
```

Incorrect:

```text
compare formatted local strings
```

Elapsed-time rules should use instants/durations.

---

# 103. "Same Day" Rule Example

Future rule:

```text
same-day cancellation has different policy
```

must define:

```text
same business date in Branch timezone
```

not:

```text
same UTC date
```

This distinction must be explicit in business requirements.

---

# 104. Age / Birthday Dates

A birthday, if stored later, is:

```text
DateOnly
```

not an instant.

Do not convert birthdays through UTC.

Same principle applies to other pure civil dates.

---

# 105. Audit vs Business Date

An Appointment may have:

```text
BusinessDate = 2026-10-05
CreatedAtUtc = 2026-09-25T11:42:00Z
```

These fields answer different questions.

Do not reuse one for the other.

---

# 106. API Naming

Use names that communicate semantics.

Preferred:

```text
startUtc
createdAtUtc
businessDate
startTime
timeZoneId
durationMinutes
```

Avoid ambiguous:

```text
date
time
timestamp
start
```

when the contract could be misunderstood.

Context-specific concise names are acceptable when semantics are documented.

---

# 107. Database Naming

Database column names should similarly preserve intent where practical:

```text
StartUtc
EndUtc
BusinessDate
TimeZoneId
```

This makes raw diagnostics safer.

---

# 108. No Manual UTC Arithmetic

Do not implement timezone conversion with:

```text
+03:30
-04:00
```

hardcoded arithmetic.

Use timezone rules.

Offsets may change historically or in future.

---

# 109. No Locale-Based Time Zone Guessing

Do not assume:

```text
fa-IR → Asia/Tehran
```

as a permanent rule.

Locale and timezone are different.

The registration UI may suggest a likely timezone, but it must be saved explicitly.

---

# 110. Reports Across Different Time Zones

If Tenant branches span time zones, cross-branch operational reporting should use:

```text
persisted BusinessDate
```

for local business-day grouping.

For event-timeline/system analytics:

```text
UTC instant
```

may be more appropriate.

The report must define its semantics.

---

# 111. Sorting

Appointment calendar sorting within one business day should use:

```text
StartUtc
```

or canonical resolved start.

Audit/event sorting uses UTC instants.

String-formatted Persian dates must never be sort keys in persistence.

---

# 112. Search Indexes

Appointment indexes should consider:

```text
(TenantId, BranchId, BusinessDate, StaffId, StartUtc)
```

or provider/query-appropriate variants.

Exact index order depends on measured query patterns.

Temporal strategy defines semantics, not a mandatory single index.

---

# 113. Appointment History

Historical display should prefer:

```text
Appointment.TimeZoneId
```

snapshot rather than blindly using the Branch's current timezone when the objective is "show what was booked locally at that time."

Product screens may explicitly offer current-zone conversion if useful.

---

# 114. User Preferred Time Zone

A future User profile may include:

```text
PreferredTimeZoneId
```

for personal display.

This does not alter:

```text
Branch schedule timezone
Appointment business timezone
```

Business operations remain anchored to Branch timezone.

---

# 115. Customer Time Zone

A public customer may reside elsewhere.

Customer-local display is optional UX.

The canonical booking remains:

```text
Branch business time
+
UTC instant
```

If shown in two timezones, both must be clearly labeled.

---

# 116. Server Logs

Technical logs should timestamp events in UTC.

Logging infrastructure may render local timestamps for operators, but stored/correlated log timestamps should remain unambiguous.

---

# 117. Cache Keys

Date-sensitive tenant/branch cache keys must use canonical date representations.

Example:

```text
tenant:{tenantId}:branch:{branchId}:appointments:2026-10-05
```

not localized display strings.

---

# 118. Idempotency Windows

Any future time-based idempotency expiration should use UTC instants.

Do not use Branch local date unless the product rule specifically defines a business-day window.

---

# 119. Definition of Done Integration

Any date/time-sensitive Story must explicitly identify whether each value is:

```text
Instant
Local Date
Local Time
Duration
Time Zone
Presentation Calendar
```

For Appointment/Schedule stories, DoD requires:

```text
No server-local time dependency
Branch timezone applied
UTC conversion verified
DateOnly/TimeOnly semantics correct
Persian display separated from storage
DST invalid/ambiguous behavior considered
Relevant temporal tests
```

---

# 120. Rejected Alternatives

Rejected:

```text
Store all dates as strings
Store Jalali date strings
Use DateTime for every temporal concept
Use server local timezone
Use browser timezone as salon timezone
Store only UTC for recurring schedules
Store only local time for Appointment instants
Use fixed numeric timezone offsets
Hardcode Asia/Tehran everywhere
Use DateTime.Now in business logic
Silently resolve ambiguous DST time
Silently shift nonexistent DST time
```

---

# 121. Consequences

## Positive

This strategy provides:

```text
Clear temporal semantics
Correct multi-timezone support
Persian UI without storage coupling
Safe Appointment comparison
Deterministic testing
Future multi-region Branch support
Stable historical booking dates
Provider-independent domain concepts
```

## Negative

It introduces:

```text
Explicit timezone configuration
Conversion service
Additional Appointment snapshot fields
DST edge-case handling
More disciplined API modeling
More temporal test cases
```

These costs are accepted because scheduling correctness depends on them.

---

# 122. Implementation Guidance

Create centralized primitives/services rather than scattered conversions.

Potential abstractions:

```text
TimeProvider
ITimeZoneService
BusinessDateResolver
```

Avoid building a large custom date framework.

Use native:

```text
DateOnly
TimeOnly
DateTimeOffset
TimeZoneInfo
```

unless concrete requirements justify another library.

---

# 123. No NodaTime Requirement

NodaTime is not required for MVP.

The selected temporal model can be implemented with modern .NET primitives.

If timezone complexity grows substantially, NodaTime may be evaluated later through a separate architecture decision.

Do not add it preemptively.

---

# 124. Example — Create Appointment

Input:

```text
Branch: Tehran Main
BusinessDate: 2026-10-05
StartTime: 10:00
Duration: 60m
TimeZone: Asia/Tehran
```

Backend:

```text
Combine DateOnly + TimeOnly
↓
Validate local time in Asia/Tehran
↓
Resolve StartUtc
↓
EndUtc = StartUtc + 60m
↓
Validate same BusinessDate
↓
Run ADR-004 concurrency flow
↓
Persist:
StartUtc
EndUtc
BusinessDate
TimeZoneId = Asia/Tehran
```

UI may display:

```text
Persian/Jalali formatted date
10:00
```

without changing persisted values.

---

# 125. Example — Schedule

Stored:

```text
DayOfWeek: Saturday
StartTime: 09:00
EndTime: 18:00
```

For:

```text
BusinessDate = 2026-10-10
Branch.TimeZoneId = Asia/Tehran
```

availability engine resolves that day's local schedule into concrete UTC intervals before comparing against Appointments.

---

# 126. Example — Dashboard Today

Request:

```text
Branch = Tehran Main
```

Backend:

```text
nowUtc = TimeProvider.GetUtcNow()
↓
convert nowUtc → Asia/Tehran
↓
businessDate = local date
↓
query Appointment.BusinessDate
```

No use of:

```text
DateTime.Today
```

on the server.

---

# 127. Example — Reminder

Appointment:

```text
StartUtc = 2026-10-05T06:30:00Z
```

Reminder policy:

```text
2 hours before
```

Worker target:

```text
2026-10-05T04:30:00Z
```

Display message can then convert Appointment time to the Branch timezone.

---

# 128. Follow-Up Decisions

This ADR directly supports:

```text
ADR-004 — Appointment Concurrency Strategy
ADR-006 — Subscription and Feature Gating
```

Potential future decisions may be required for:

```text
Persian reporting periods
overnight appointments
customer-local timezone display
timezone-change migration policy
advanced recurring schedules
```

Only create additional ADRs when those become real requirements.

---

# 129. Final Decision Summary

Shinera distinguishes:

```text
Global Instant
Business Local Date/Time
Presentation Calendar
```

Canonical rules:

```text
Global instants
→ DateTimeOffset normalized to UTC

Business dates
→ DateOnly

Business wall times
→ TimeOnly

Time zones
→ IANA identifiers

Recurring schedules
→ local wall time + Branch timezone

Appointments
→ StartUtc + EndUtc + BusinessDate + TimeZoneId snapshot

Persian/Jalali
→ UI presentation/input only

Current time
→ TimeProvider

Server local time
→ never authoritative
```

Tenant has:

```text
DefaultTimeZoneId
```

Branch has:

```text
TimeZoneId
```

and Branch timezone is authoritative for operational scheduling.

Local appointment input is resolved server-side into an unambiguous UTC instant.

Invalid and ambiguous DST local times are rejected rather than silently guessed.

This strategy is the temporal foundation for Staff Schedule, Appointment Availability, Concurrency, Dashboard, Reporting, and Notifications.

---

# 130. Official Technical References

- .NET dates, times, and time zones:
  https://learn.microsoft.com/dotnet/standard/datetime/

- `DateTimeOffset`:
  https://learn.microsoft.com/dotnet/api/system.datetimeoffset

- `DateOnly` / `TimeOnly` support in EF Core:
  https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/whatsnew

- `TimeProvider`:
  https://learn.microsoft.com/dotnet/standard/datetime/timeprovider-overview

- Testing with `FakeTimeProvider`:
  https://learn.microsoft.com/dotnet/core/extensions/timeprovider-testing

- `TimeZoneInfo` IANA/Windows conversion:
  https://learn.microsoft.com/dotnet/api/system.timezoneinfo.tryconvertianaidtowindowsid

- Npgsql date/time mapping:
  https://www.npgsql.org/efcore/release-notes/6.0.html
