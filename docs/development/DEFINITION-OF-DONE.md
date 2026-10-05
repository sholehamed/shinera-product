# Shinera — Definition of Done

**Project:** Shinera  
**Document Role:** Completion and quality gate for backlog work  
**Audience:** Developers, reviewers, ChatGPT, Work, coding agents, QA  
**Status:** Active  
**Version:** 1.0

---

# 1. Purpose

This document defines when work in Shinera may legitimately be marked:

```text
Done
```

A story is **not Done** merely because:

- code was written,
- an endpoint exists,
- a page renders,
- a migration was generated,
- the happy path works locally,
- frontend and backend compile independently,
- an AI agent reports completion.

A story is Done only when the applicable product, technical, security, integration, testing, and UX requirements are satisfied.

---

# 2. Core Rule

For every backlog story:

```text
Every DoD area must be evaluated.

Applicable → must pass.
Not Applicable → explicitly mark N/A.
Unknown → story is not Done.
```

Do not silently skip a section because it appears irrelevant.

---

# 3. Status Model

Recommended story lifecycle:

```text
Not Started
    ↓
In Progress
    ↓
Backend Done / Frontend Done
    ↓
Integration Pending
    ↓
Testing
    ↓
Review
    ↓
Done
```

Alternative status when necessary:

```text
Blocked
```

`Done` is the terminal state for the approved story scope.

---

# 4. Minimum Definition of Done

A story cannot be marked Done unless all applicable items below pass:

```text
✓ Approved scope implemented
✓ Acceptance criteria satisfied
✓ Business rules enforced
✓ Backend authorization enforced
✓ Tenant isolation verified
✓ Validation implemented
✓ Database changes completed if needed
✓ Frontend states handled if needed
✓ Backend/frontend contract integrated
✓ Relevant automated tests added
✓ Build succeeds
✓ Relevant tests pass
✓ No known Critical/High defect remains
✓ Documentation/status updated when required
```

---

# 5. Product Completion

## Required

- Story implementation matches the approved backlog scope.
- Acceptance criteria are satisfied.
- No unapproved features were added.
- No important requirement was silently changed.
- Edge cases identified during implementation are either:
  - implemented,
  - explicitly deferred,
  - or recorded as a follow-up story.

## Product Decision Gate

If implementation required changing any of the following, the change must have been reviewed as a product decision:

- MVP scope
- Appointment behavior
- Registration flow
- Subscription behavior
- Plan capabilities
- Roles or permissions
- Cancellation behavior
- VIP behavior
- Force Appointment behavior
- Public booking flow

A developer or agent must not hide product decisions inside implementation details.

---

# 6. Business Rules

Business rules must be enforced in a trustworthy layer.

Frontend-only enforcement is never sufficient for critical rules.

Examples:

```text
Appointment conflict
Appointment transition
Staff availability
Tenant ownership
Branch ownership
Subscription entitlement
Permission scope
```

## Done Criteria

- Rules are implemented in backend/domain/application layer as appropriate.
- Invalid transitions or invalid business states are rejected.
- Stable error codes exist where frontend behavior depends on the failure.
- Business rules are covered by automated tests when practical.

---

# 7. Backend Definition of Done

Applicable when the story changes backend behavior.

## Architecture

- Existing Shinera architecture is followed.
- Existing patterns are reused when suitable.
- No unnecessary new abstraction or package is introduced.
- Endpoint remains thin.
- Business logic is not buried in API endpoints.

## CQRS

Where applicable:

```text
Command → mutation
Query   → read
```

- Command/query responsibility is clear.
- Handler is focused.
- Current `IDispatcher` convention is used.
- No competing dispatcher pattern is introduced.

## Result Pattern

Expected failures use the current project Result pattern.

Examples:

```text
validation
not found
conflict
business rule violation
feature unavailable
access denied
```

Unexpected infrastructure failures may use exception handling according to current project conventions.

## API

- Route follows existing conventions.
- Request DTO is intentional.
- Response DTO is intentional.
- API response envelope follows current project convention.
- HTTP status code is correct.
- Error code is stable and machine-readable where needed.
- Scalar/OpenAPI metadata remains valid.

---

# 8. Multi-Tenancy Definition of Done

Applicable to every tenant-owned feature.

This is a **critical security gate**.

## Required

- Tenant-specific entity ownership is explicit.
- `TenantId` supplied by the client is not trusted as authorization.
- Current tenant context is used.
- Related entities are validated against the current tenant.
- Cross-tenant IDs cannot be used to bypass isolation.
- Reads are tenant-scoped.
- Writes are tenant-scoped.
- Deletes/deactivation are tenant-scoped.

## Required Tests for Critical Features

At least the relevant forms of:

```text
Tenant A cannot read Tenant B data
Tenant A cannot update Tenant B data
Tenant A cannot delete Tenant B data
Tenant A cannot reference Tenant B data
```

## Blocking Rule

Any known tenant isolation issue means:

```text
Story != Done
```

regardless of all other checks.

---

# 9. Branch Isolation Definition of Done

Applicable when Branch matters.

- Branch belongs to current Tenant.
- User has access to the Branch.
- Current Branch semantics are respected where applicable.
- Branch-scoped resources cannot be accessed through arbitrary IDs.
- Main Branch invariants remain valid.
- Multi-branch feature entitlement is enforced where applicable.

---

# 10. Authorization Definition of Done

Applicable to protected functionality.

## Backend

- Correct permission is defined or reused.
- Correct action is enforced.
- Correct scope is enforced.
- Unauthorized requests fail server-side.
- Frontend hiding is not treated as security.

## Frontend

Where applicable:

- Unauthorized actions are hidden or disabled.
- Direct navigation still relies on backend security.
- UI responds correctly to `Forbidden` / access errors.

## Tests

At minimum for security-sensitive operations:

```text
authorized user → allowed
unauthorized user → denied
wrong scope → denied
```

---

# 11. Subscription / Feature Gating Definition of Done

Applicable to gated features.

## Required

Frontend:

```text
Unavailable feature
→ hidden / disabled / upgrade state
```

Backend:

```text
Unavailable feature
→ request rejected
```

## Rules

- Feature capability is preferred over hardcoded plan-name checks.
- Feature behavior is consistent across UI and API.
- Entitlement rules are testable.
- Bypassing frontend must not bypass entitlement.

---

# 12. Database Definition of Done

Applicable when persistence changes.

## Model

- Entity/model updated.
- EF configuration updated.
- Relationships are intentional.
- Nullability is intentional.
- Cascade behavior is intentional.

## Indexes

Relevant indexes are considered for:

```text
TenantId
BranchId
foreign keys
search fields
unique identifiers
high-frequency filters
```

Do not add indexes mechanically.

## Constraints

Database constraints are added when they safely strengthen integrity.

Examples:

- unique constraints
- required relationships
- valid ownership relations

## Migration

- Migration exists.
- Migration has been reviewed.
- Up migration is valid.
- Down/revert behavior is understood where relevant.
- Existing data impact is considered.
- Application can start against the migrated schema.

## Blocking Rule

A model change without the required migration is not Done.

---

# 13. Query and Performance Definition of Done

Applicable to queries and list endpoints.

## Required

- No obvious N+1 behavior.
- No accidental full-table load.
- Projection is used when appropriate.
- Pagination exists for potentially large collections.
- Filters are applied server-side where appropriate.
- Search/sort behavior is deterministic.
- Unnecessary `Include` chains are avoided.

## Performance Review

Performance optimization is evidence-based.

Do not require:

```text
CompiledQuery everywhere
Caching everywhere
Premature denormalization
```

A story is not blocked on theoretical optimization unless a measurable risk exists.

---

# 14. Appointment-Specific Definition of Done

Applicable to Appointment stories.

Because Appointments are the core Shinera domain, these stories require stricter checks.

## Creation / Reschedule

Validate where applicable:

```text
Tenant
Branch
Customer
Service
Staff
Staff active
Service active
Staff ↔ Service assignment
Staff ↔ Branch assignment
Schedule
Breaks
Time off
Existing appointments
Duration
State
Permissions
Feature entitlement
```

## Conflict

Backend performs final conflict validation.

Frontend available slots are not authoritative.

## Concurrency

For slot reservation:

- Concurrent booking risk has been considered.
- The selected concurrency strategy is enforced.
- A race condition must not knowingly allow duplicate occupation of the same staff slot.

## State Transition

Illegal transitions are rejected.

Examples:

```text
Confirmed → InProgress      valid
InProgress → Completed      valid
Confirmed → Cancelled       valid

Completed → InProgress      invalid
```

unless a specific approved business rule says otherwise.

## Tests

Appointment-critical stories should include appropriate coverage for:

```text
overlap
adjacent slot
cancelled appointment
different staff
schedule boundaries
breaks
invalid state transition
tenant isolation
permission
concurrency where feasible
```

---

# 15. Date and Time Definition of Done

Applicable to date/time-sensitive features.

- Persistence and display concerns are separated.
- Persian formatted dates are not stored as domain date values.
- Gregorian/UTC strategy is respected where applicable.
- Timezone behavior is explicit when it affects business rules.
- Date conversion is not duplicated inconsistently across the application.
- Boundary cases are considered.

Examples:

```text
day boundary
timezone conversion
working-hour boundary
cross-midnight rules if supported
```

---

# 16. Frontend Definition of Done

Applicable when the story changes UI.

## Functional

- Route works.
- API integration works.
- Form submission works.
- Server validation errors are handled.
- Business errors are handled.
- Permissions are reflected correctly.
- Feature gating is reflected correctly.

## Required UI States

Evaluate:

```text
Loading
Empty
Success
Validation Error
API Error
Unauthorized / Forbidden
Feature Unavailable
```

Use `N/A` only when genuinely irrelevant.

## Visual

- Matches current Shinera design language.
- Uses Angular Material / Trezo conventions.
- Avoids unnecessary new UI library.
- Existing design tokens/theme variables are reused.

## RTL

Check:

- layout direction
- spacing
- icon placement
- directional arrows
- menus
- forms
- tables
- pagination
- date controls

## Theme

- Light mode checked.
- Dark mode checked.

## Responsive

At minimum verify the page remains usable on:

```text
Desktop
Tablet / narrow desktop
Mobile where the feature is expected to be used
```

Pixel perfection is not required for all breakpoints, usability is.

---

# 17. Forms Definition of Done

Applicable to forms.

- Required fields are marked and enforced.
- Client validation improves UX.
- Server validation remains authoritative.
- Submit is protected against accidental duplicate submission where relevant.
- Loading state exists during submission.
- Success behavior is clear.
- Error behavior is clear.
- User-entered data is not unnecessarily lost after recoverable errors.
- Reset/cancel behavior is intentional.

---

# 18. API Contract Definition of Done

Applicable to cross-stack features.

Backend and frontend agree on:

```text
Route
Method
Request
Response
Enums
Nullable fields
Validation errors
Business errors
Pagination
Date format
Money format
Permission behavior
```

Generated clients or contract tooling must be regenerated if the project workflow requires it.

A backend feature is not considered fully integrated if the frontend still depends on mock data for the same approved story.

---

# 19. Error Handling Definition of Done

## Required

- Business errors have stable codes where needed.
- Frontend does not parse human-readable message text to determine logic.
- Unexpected exceptions do not expose internal stack traces in production.
- Errors are meaningful enough for troubleshooting.
- Sensitive information is not included in error responses.

Recommended error code style:

```text
appointment.conflict
appointment.invalid_transition
staff.not_available
subscription.feature_unavailable
tenant.access_denied
```

---

# 20. Testing Definition of Done

Testing depth depends on risk.

Not every story needs every test category.

Every story must consciously determine which categories apply.

## Domain Tests

Use for:

- invariants
- transitions
- calculations
- pure business rules

## Application Tests

Use for:

- command/query behavior
- validation
- Result behavior
- orchestration

## Integration Tests

Use for:

- EF behavior
- tenant isolation
- authorization
- persistence
- transactions
- real endpoint/application infrastructure
- critical business workflows

## E2E Tests

Use for important user journeys.

Primary MVP golden path:

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

Registration should have separate E2E coverage.

---

# 21. Test Quality Rules

Automated tests should:

- test behavior, not implementation trivia,
- have deterministic setup,
- avoid dependency on test execution order,
- clearly communicate failure,
- include meaningful negative cases,
- avoid meaningless coverage-only tests.

A test that only proves a mocked method returns what the mock was told to return provides little value.

---

# 22. Build Verification

Before Done:

## Backend changes

```text
dotnet build
```

must succeed for the affected solution/projects.

Relevant automated tests must pass.

## Frontend changes

Frontend production build must succeed.

Relevant unit/E2E tests must pass where applicable.

## Cross-stack changes

Both sides must build.

Integration must be exercised.

---

# 23. Security Definition of Done

For security-sensitive changes, consider:

```text
Authentication
Authorization
Tenant isolation
Branch isolation
Token lifecycle
Input validation
Rate limiting
Public endpoint abuse
Sensitive logging
File upload risk
Data exposure
```

## Forbidden Logging

Do not log:

```text
Passwords
Access tokens
Refresh tokens
Secrets
Raw sensitive credentials
```

Known Critical or High security defects block Done.

---

# 24. Observability Definition of Done

Applicable to meaningful backend operations.

Where supported:

- Failures are diagnosable.
- Useful request context is logged.
- Trace correlation is preserved.
- Sensitive data is excluded.
- Important unexpected failures surface through current monitoring/logging infrastructure.

Do not add excessive noisy logs merely to satisfy this section.

---

# 25. Accessibility Definition of Done

Applicable to user-facing UI.

At minimum:

- controls are keyboard reachable where expected,
- form controls have meaningful labels,
- icon-only actions have accessible meaning,
- disabled states are understandable,
- contrast is not obviously broken,
- interaction does not depend solely on color where practical.

Full certification is not required for MVP unless separately scoped.

---

# 26. Code Quality Definition of Done

Code should be:

```text
Readable
Focused
Consistent
Maintainable
Predictable
```

Avoid:

- unrelated refactors
- dead code
- large commented-out blocks
- TODOs that hide required functionality
- magic strings for business-critical values
- duplicated core business rules
- unnecessary generic abstractions
- premature framework-building

---

# 27. Dependency Definition of Done

If a new package is introduced:

- necessity is justified,
- current project capabilities were checked first,
- package is actively maintained enough for the use case,
- competing package for the same concern is not already present,
- security/licensing concerns are acceptable,
- package impact is understood.

Significant infrastructure dependencies should be documented.

---

# 28. Documentation Definition of Done

Update documentation when the story changes:

- public API behavior,
- architecture,
- product behavior,
- developer workflow,
- configuration,
- deployment,
- environment variables,
- important operational behavior.

Not every code change requires documentation.

## Required Project Updates

Update `DEVELOPMENT-STATUS.md` when:

- story starts,
- major phase changes,
- backend/frontend split status changes,
- story becomes blocked,
- story becomes Done.

Create/update ADR when required by `PROJECT-INSTRUCTIONS.md`.

---

# 29. Git / Change Scope Definition of Done

Changes should be focused.

A Story should not contain unrelated broad refactors.

Reviewable change sets are preferred.

Before completion:

- accidental generated files are excluded,
- secrets are not committed,
- debug artifacts are removed,
- obsolete temporary files are removed,
- migration files are intentional.

---

# 30. Review Definition of Done

Important stories should receive a review appropriate to risk.

Review dimensions:

```text
Product correctness
Business rules
Tenant isolation
Authorization
Feature gating
Data integrity
Concurrency
Performance
Error handling
Test coverage
Maintainability
UX states
RTL / Theme
```

Findings may be classified:

```text
Critical
High
Medium
Low
```

## Blocking

Open:

```text
Critical
High
```

findings block Done.

`Medium` findings should normally be resolved or explicitly deferred.

`Low` findings may be deferred if they do not affect acceptance criteria or safety.

---

# 31. Agent Completion Rule

An AI coding agent may report:

```text
Implementation Complete
```

only if it can state what was actually verified.

Required report:

```text
Implemented:
- ...

Build:
- command
- result

Tests:
- command
- result

Not Verified:
- ...

Remaining Risks:
- ...
```

An agent must never claim:

```text
tests passed
build passed
migration works
```

unless those checks were actually executed.

If execution was impossible, report:

```text
Not Verified
```

not `Done`.

---

# 32. Partial Completion Rule

Sometimes a cross-stack story cannot be completed in one pass.

Use explicit intermediate states.

Example:

```text
SHN-132 Create Appointment

Overall: In Progress

Database: Done
Backend: Done
Frontend: In Progress
Authorization: Done
Tests: Partial
E2E: Not Started
```

Do not mark the overall Story Done.

---

# 33. Backend-Only Story Rule

A genuinely backend-only story can be Done without frontend work.

Frontend:

```text
N/A
```

must be intentional.

Examples:

- infrastructure health endpoint
- internal interceptor
- database migration tooling
- backend-only security hardening

---

# 34. Frontend-Only Story Rule

A genuinely frontend-only story can be Done without backend changes when it does not require a new API/business rule.

Backend:

```text
N/A
```

Examples:

- visual layout correction
- spacing
- dark mode style fix
- static content correction

If the UI creates a new business behavior, it is not truly frontend-only.

---

# 35. Spike / Research Rule

Research tasks and spikes use a different completion condition.

A spike is Done when:

- question is clearly answered,
- evidence is recorded,
- alternatives are identified,
- recommendation/decision is documented,
- unresolved risks are listed,
- no production implementation is falsely implied.

Spike output may become:

```text
ADR
Implementation Story
Product Decision
Technical Note
```

---

# 36. Bug Definition of Done

A bug fix is Done when:

- root cause is understood enough to make the fix safe,
- defect is fixed,
- regression test is added where practical,
- related behavior is not broken,
- build succeeds,
- applicable tests pass,
- status/incident docs updated if needed.

For security or tenant isolation bugs, regression tests are mandatory unless technically impossible.

---

# 37. Hotfix Rule

Urgency may reduce process overhead but not core safety.

Hotfix may skip:

- broad refactoring,
- optional documentation,
- noncritical cleanup.

Hotfix must not skip:

```text
Build
Relevant validation
Tenant isolation
Authorization
Critical regression check
```

Follow-up cleanup may be created separately.

---

# 38. Waiver / Exception Rule

A Story may occasionally ship with an unmet noncritical DoD item.

This requires an explicit recorded exception.

Record:

```text
Missing Item:
Reason:
Risk:
Owner:
Follow-up Story:
Target:
```

Examples:

```text
Missing Item: Playwright test
Reason: Temporary CI browser infrastructure failure
Risk: Manual regression required
Follow-up: SHN-XXX
```

## Cannot Be Waived Casually

These require explicit high-confidence approval and normally block release:

```text
Tenant isolation
Authorization
Known data corruption
Known duplicate appointment race
Credential exposure
Critical security defect
Broken production build
```

---

# 39. Release-Level Definition of Done

A Story being Done does not automatically mean the Release is ready.

For MVP release:

```text
✓ MVP Exit Criteria satisfied
✓ Critical golden-path E2E passes
✓ Tenant isolation suite passes
✓ Authentication flow verified
✓ Registration flow verified
✓ Appointment conflict protection verified
✓ Payment flow verified
✓ Production builds succeed
✓ Database migration path verified
✓ Required configuration documented
✓ No open Critical/High defects
✓ Release candidate reviewed
```

---

# 40. MVP Golden Path Gate

Before MVP may be called usable:

```text
Owner selects plan
→ registers
→ Tenant is created
→ Main Branch is created
→ Owner enters workspace
→ Service is created
→ Staff is created
→ Service is assigned
→ Schedule is configured
→ Customer is created
→ Available slot is calculated
→ Appointment is created
→ Duplicate/conflicting booking is prevented
→ Appointment starts
→ Appointment completes
→ Payment is registered
→ Dashboard reflects real data
```

This flow must work using real integrated backend and frontend behavior, not mocked dashboard data.

---

# 41. Story Completion Template

Use this template when marking a story complete:

```text
Story: SHN-XXX — <Title>

Status: Done

Scope:
✓ Acceptance criteria satisfied

Backend:
✓ Done / N/A

Frontend:
✓ Done / N/A

Database:
✓ Done / N/A

Tenant Isolation:
✓ Verified / N/A

Authorization:
✓ Verified / N/A

Feature Gate:
✓ Verified / N/A

Tests:
✓ Domain: ...
✓ Application: ...
✓ Integration: ...
✓ E2E: ... / N/A

Build:
✓ Backend
✓ Frontend

Documentation:
✓ Updated / N/A

Known Deferred Items:
- None

Remaining Critical/High Defects:
- None
```

---

# 42. Pull Request Completion Checklist

Recommended PR checklist:

```text
## Product
- [ ] Story / requirement linked
- [ ] Acceptance criteria satisfied
- [ ] No unapproved scope added

## Backend
- [ ] Business rules implemented
- [ ] Validation implemented
- [ ] Error codes/statuses correct
- [ ] API contract reviewed

## Security
- [ ] Tenant isolation checked
- [ ] Authorization checked
- [ ] Branch scope checked
- [ ] Feature entitlement checked
- [ ] Sensitive data not logged

## Database
- [ ] Model/configuration updated
- [ ] Indexes/constraints considered
- [ ] Migration reviewed

## Frontend
- [ ] Loading state
- [ ] Empty state
- [ ] Error state
- [ ] Permissions
- [ ] RTL
- [ ] Dark mode
- [ ] Responsive behavior

## Testing
- [ ] Relevant automated tests added
- [ ] Backend build passed
- [ ] Frontend build passed
- [ ] Relevant test suites passed
- [ ] E2E added/run when required

## Documentation
- [ ] DEVELOPMENT-STATUS updated
- [ ] ADR/docs updated if required

## Final
- [ ] No open Critical/High defect
```

---

# 43. Definition of Ready vs Definition of Done

Do not confuse the two.

## Ready

A story is Ready when there is enough clarity to start implementation.

Typical requirements:

```text
Goal known
Scope known
Acceptance criteria known
Dependencies understood
Major product ambiguity resolved
```

## Done

A story is Done only after implementation, integration, verification, and applicable quality gates pass.

A separate `DEFINITION-OF-READY.md` may be added later if backlog flow requires it.

---

# 44. Final Rule

The purpose of Definition of Done is not to create paperwork.

It exists to prevent this:

```text
"It works on my page."
"It builds."
"The endpoint returns 200."
"The UI is finished."
"The agent said it is complete."
```

from being confused with:

```text
Production-worthy feature behavior.
```

For Shinera:

```text
Done =
Correct
+ Secure
+ Tenant-safe
+ Integrated
+ Tested
+ Reviewable
+ Usable
```

If any critical part is unknown, the Story is not Done.
