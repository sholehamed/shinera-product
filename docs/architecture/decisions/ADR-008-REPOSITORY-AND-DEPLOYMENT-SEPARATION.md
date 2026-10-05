# ADR-008 — Repository and Deployment Separation

**Project:** Shinera  
**Status:** Accepted  
**Date:** 2026-10-05  
**Decision Type:** Architecture / Repository Strategy / Delivery  
**Scope:** Product Documentation, Backend, Frontend, CI/CD, Release Management  
**Related Documents:**
- `SHINERA-PRD.md`
- `SHINERA-PRODUCT-BACKLOG.md`
- `PROJECT-INSTRUCTIONS.md`
- `DEVELOPMENT-STATUS.md`
- `DEFINITION-OF-DONE.md`
- `ARCHITECTURE-OVERVIEW.md`
- `ADR-001-MULTI-TENANCY-STRATEGY.md`
- `ADR-002-AUTHENTICATION-WITH-OPENIDDICT.md`
- `ADR-003-AUTHORIZATION-AND-PERMISSION-MODEL.md`

---

# 1. Context

Shinera consists of three distinct concerns:

```text
Product / Architecture
Backend
Frontend
```

The implementation already has naturally independent technology stacks:

```text
Backend
→ .NET 10
→ ASP.NET Core
→ EF Core
→ OpenIddict
→ Minimal API
→ Integration Tests

Frontend
→ Angular 20
→ Angular Material
→ Trezo
→ Playwright
```

Product and architecture documentation also have a lifecycle independent from application builds.

The repository strategy must support:

- independent backend/frontend development,
- independent deployment,
- clear ownership,
- reliable product documentation,
- cross-repository planning,
- simple CI/CD,
- minimal duplication,
- future scaling of the team/project.

---

# 2. Decision

Shinera will use three separate repositories:

```text
shinera-product
shinera-backend
shinera-frontend
```

Responsibilities:

```text
shinera-product
→ product, architecture, roadmap, decisions, development governance

shinera-backend
→ backend application, persistence, APIs, security, backend tests

shinera-frontend
→ Angular application, UI, client integration, frontend tests
```

Backend and frontend will be independently buildable and deployable.

Shinera will not use a monorepo for the primary application at this stage.

---

# 3. Repository Overview

```text
Shinera
│
├── shinera-product
│
├── shinera-backend
│
└── shinera-frontend
```

These repositories together form one product.

Repository separation does not mean architectural isolation.

They remain coordinated through:

```text
Product Backlog
Architecture Decisions
API Contracts
Release Process
Cross-repo Stories
```

---

# 4. shinera-product Responsibility

`shinera-product` is the project control-plane repository.

It owns:

```text
Product Definition
Product Backlog
Architecture Documentation
Architecture Decision Records
Development Status
Definition of Done
Development Instructions
Roadmap
Cross-repository decisions
```

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
            ├── ADR-001-...
            ├── ADR-002-...
            └── ...
```

---

# 5. shinera-product Is Not an Application Repository

Do not place production application code in:

```text
shinera-product
```

Avoid:

```text
Angular source
.NET source
database migrations
deployment binaries
generated API clients
```

unless a future tooling-specific folder is intentionally added.

The product repo remains lightweight and documentation-oriented.

---

# 6. shinera-backend Responsibility

`shinera-backend` owns:

```text
Domain
Application
Infrastructure
API
Database
Migrations
Authentication
Authorization
Subscription enforcement
Backend integration
Backend tests
```

Typical structure:

```text
shinera-backend/
├── src/
│   ├── Domain
│   ├── Application
│   ├── Infrastructure
│   └── Api
│
├── tests/
│   ├── Application.Tests
│   └── IntegrationTests
│
└── ...
```

Exact folder layout follows the real implementation.

---

# 7. shinera-frontend Responsibility

`shinera-frontend` owns:

```text
Angular application
Routes
Pages
Components
Forms
Theme
RTL
Localization presentation
API integration
Permission UX
Feature-gating UX
Playwright
Frontend tests
```

Typical structure:

```text
shinera-frontend/
├── src/
├── e2e/
├── angular.json
├── package.json
└── ...
```

---

# 8. Why Separate Repositories

The selected separation provides:

```text
Independent releases
Independent CI pipelines
Independent rollback
Smaller repository scope
Clear technology ownership
Cleaner automation
Simpler frontend/backend deployment
```

It also matches Shinera's current project organization and development workflow.

---

# 9. Why Monorepo Is Rejected

A monorepo was considered:

```text
shinera/
├── backend/
├── frontend/
└── docs/
```

Advantages:

- single commit can change both stacks,
- simple unified history,
- easier atomic contract changes.

However, current disadvantages outweigh those benefits:

- backend/frontend deployment lifecycles are separate,
- technology stacks are independent,
- CI becomes more coupled,
- repository grows in scope,
- frontend/backend ownership boundaries become less clear.

Decision:

```text
No monorepo for current Shinera architecture.
```

This may be revisited only if cross-stack coordination cost becomes materially higher than repository separation benefits.

---

# 10. Product Documentation Duplication Is Rejected

Canonical documents must not be copied into:

```text
shinera-backend
and
shinera-frontend
```

Examples:

```text
SHINERA-PRD.md
SHINERA-PRODUCT-BACKLOG.md
ARCHITECTURE-OVERVIEW.md
ADRs
```

must remain canonical in:

```text
shinera-product
```

Duplicating them creates drift.

---

# 11. Repository README References

Backend and frontend README files should link to the product repository.

Conceptually:

```text
For product requirements, architecture decisions, and backlog:
see sholehamed/shinera-product
```

They may include short implementation-specific summaries.

They should not reproduce the entire PRD or architecture documents.

---

# 12. Local Repository Arrangement

For local development, repositories may be checked out side-by-side:

```text
shinera/
├── shinera-product/
├── shinera-backend/
└── shinera-frontend/
```

This is a local workspace organization only.

It does not make them one Git repository.

---

# 13. Git Independence

Each repository has its own:

```text
Git history
Branches
Pull Requests
Tags
Releases
CI pipeline
```

A frontend release does not require a backend commit when no backend change is required.

A backend release does not require a frontend release when the contract remains compatible.

---

# 14. Cross-Repository Story

One backlog Story may require changes in both repositories.

Example:

```text
SHN-132 — Create Appointment
```

may involve:

```text
shinera-backend
→ command
→ validation
→ endpoint
→ tests

shinera-frontend
→ form
→ API integration
→ UX states
→ E2E
```

The Story remains one product Story even though implementation spans repositories.

---

# 15. Story Tracking

Product backlog identity is global.

Example:

```text
SHN-132
```

must mean the same feature across:

```text
product
backend
frontend
```

Repository-specific issues/PRs should reference the same Story ID.

---

# 16. GitHub Issues

Recommended issue strategy:

```text
Product Story
→ canonical Story ID

Backend issue
→ SHN-132 Backend

Frontend issue
→ SHN-132 Frontend
```

when separate implementation tracking is useful.

For small Stories, one cross-repository planning item may be enough.

---

# 17. GitHub Project Board

A GitHub Project board may aggregate:

```text
shinera-product
shinera-backend
shinera-frontend
```

issues into one delivery view.

This provides one project-management surface without requiring a monorepo.

---

# 18. Pull Request Naming

Recommended PR naming:

```text
SHN-132 Create appointment backend
SHN-132 Create appointment UI
```

or equivalent.

The Story ID keeps cross-repo work discoverable.

---

# 19. Commit Scope

Commits should remain repository-focused.

Avoid unrelated cross-feature changes.

Example backend commit:

```text
feat(appointments): implement SHN-132 create appointment
```

Frontend:

```text
feat(appointments): integrate SHN-132 create appointment
```

Exact Conventional Commit usage is optional but consistency is recommended.

---

# 20. Backend / Frontend Contract

The primary integration boundary is:

```text
HTTP API Contract
```

This includes:

```text
Routes
Methods
Request DTOs
Response DTOs
Enums
Error Codes
Pagination
Authentication behavior
Date/time formats
```

Repositories must not depend on shared source-code types.

---

# 21. Shared Source Project Is Rejected

Do not create:

```text
shared-models
```

Git submodule/package merely to share C# DTOs with TypeScript.

The technologies and type systems differ.

The API contract is the source for generated/client types.

---

# 22. Generated API Client

Where practical, frontend client types may be generated from backend OpenAPI/Scalar contract.

Conceptually:

```text
Backend OpenAPI
↓
Client Generator
↓
Angular TypeScript client/models
```

Generated code belongs to:

```text
shinera-frontend
```

or is generated during its build workflow.

---

# 23. Generated Code Ownership

Generated frontend API code should not be manually edited.

If the contract is wrong:

```text
fix backend contract
↓
regenerate
```

Do not patch generated models manually as a permanent solution.

---

# 24. Contract Compatibility

Backend should avoid unnecessary breaking API changes.

Examples of breaking changes:

```text
renaming property
removing field
changing enum semantics
changing route
changing error meaning
```

When breaking changes are necessary, frontend coordination is required.

---

# 25. API Versioning

Repository separation does not require immediate API versioning.

Shinera may evolve one internal first-party API while frontend/backend are coordinated.

Formal API versioning should be introduced when:

```text
multiple client versions must coexist
external consumers exist
backward compatibility becomes operationally required
```

Do not version prematurely.

---

# 26. Deployment Independence

Backend and frontend are independently deployable.

Conceptually:

```text
Frontend Deployment
        │
        │ HTTPS
        ▼
Backend Deployment
        │
        ▼
Database
```

Each may have separate:

```text
build pipeline
artifact
deployment
rollback
```

---

# 27. Deployment Coupling

Independent deployment does not mean incompatible versions may be released carelessly.

Each release must consider:

```text
API compatibility
database migrations
frontend expectations
authentication configuration
environment configuration
```

---

# 28. Backward-Compatible Deployment Order

For additive compatible changes, preferred rollout is often:

```text
Backend first
↓
Frontend second
```

Example:

```text
Backend adds optional field/endpoint
↓
Frontend begins using it
```

This minimizes frontend calling an endpoint that does not yet exist.

---

# 29. Breaking Deployment

If a breaking contract change cannot be avoided:

```text
introduce compatibility bridge
or
coordinate release window
or
temporary parallel contract
```

Do not deploy a frontend that immediately breaks against the currently deployed backend.

---

# 30. Database Migrations

Database migrations belong exclusively to:

```text
shinera-backend
```

Frontend never manages schema migrations.

Backend deployment must own:

```text
migration generation
migration review
migration execution strategy
rollback planning
```

---

# 31. Migration Compatibility

When zero/minimal downtime matters, schema changes should consider deployment ordering.

Prefer additive migrations such as:

```text
Add nullable/new column
Deploy compatible backend
Backfill if needed
Later enforce constraint/remove old field
```

rather than immediate destructive changes.

---

# 32. Product Repo Deployment

`shinera-product` has no runtime deployment.

It may have:

```text
documentation validation
Markdown lint
link checking
GitHub Pages/docs publication
```

later if useful.

It is not part of production runtime.

---

# 33. Backend CI

Target pipeline:

```text
Restore
↓
Build
↓
Application/Unit Tests
↓
Integration Tests
↓
Publish
```

Optional future stages:

```text
container build
security scan
migration validation
deployment
```

---

# 34. Frontend CI

Target pipeline:

```text
Install
↓
Lint / validation
↓
Build
↓
Unit tests
↓
Playwright
↓
Publish
```

Exact stages depend on project tooling.

---

# 35. Product CI

Optional lightweight pipeline:

```text
Markdown validation
broken-link checks
ADR filename checks
documentation structure checks
```

No heavy application pipeline is required.

---

# 36. Release Artifacts

Backend artifact may be:

```text
.NET publish output
Docker image
```

Frontend artifact may be:

```text
static Angular build
container
CDN artifact
```

Exact hosting model is deployment-specific.

---

# 37. Versioning

Backend and frontend may have independent technical version numbers.

Example:

```text
Backend v1.8.0
Frontend v1.12.0
```

Product release may separately be called:

```text
Shinera MVP
Shinera 1.0
```

Do not require repository package versions to match numerically.

---

# 38. Release Compatibility Matrix

If independent release cadence becomes frequent, maintain a lightweight compatibility rule.

Example:

```text
Frontend >= 1.12
requires
Backend >= 1.8
```

This is optional until operationally necessary.

---

# 39. Environment Configuration

Backend and frontend each own their configuration surfaces.

Backend examples:

```text
Connection strings
OpenIddict
Signing certificates
CORS
Logging
Storage
Notification providers
```

Frontend examples:

```text
API base URL
environment flags
public configuration
```

Secrets must remain backend/server-side.

---

# 40. Frontend Must Not Receive Backend Secrets

Never expose:

```text
database credentials
client secrets
signing keys
private API secrets
```

through Angular environment files.

Anything shipped in the browser is public.

---

# 41. CORS

Because frontend and backend may deploy separately, production CORS configuration must explicitly allow approved frontend origins.

This belongs to backend environment configuration.

CORS does not replace authentication/authorization.

---

# 42. Authentication Redirect URIs

ADR-002 OpenIddict client redirect URIs depend on frontend deployment URLs.

Therefore frontend environment/release changes affecting:

```text
origin
callback URI
logout URI
```

must coordinate with backend OpenIddict configuration.

This is an explicit cross-repository contract.

---

# 43. Shared Environment Naming

Repositories should use consistent environment terminology.

Recommended:

```text
Development
Test
Staging
Production
```

Avoid one repository calling an environment:

```text
preprod
```

while another calls the same environment:

```text
stage
```

unless intentionally mapped/documented.

---

# 44. Development Environment

Local development may run:

```text
Angular localhost
+
Backend localhost
+
Local/container database
```

Frontend API URL points to local backend.

Backend CORS permits the local frontend origin.

---

# 45. Integration Environment

A shared staging environment should validate:

```text
deployed backend
deployed frontend
real database provider
authentication
cross-stack E2E
```

This catches issues that independent builds cannot.

---

# 46. End-to-End Tests

Playwright belongs primarily to:

```text
shinera-frontend
```

because tests drive the user-facing application.

However they validate:

```text
frontend
+
backend
+
database
```

as an integrated system.

CI may run them against a composed test environment.

---

# 47. Backend Integration Tests

Backend repository owns API/database integration tests.

These should not require Angular.

Examples:

```text
Tenant isolation
Authentication
Authorization
Appointment concurrency
Persistence
```

---

# 48. Cross-Repository Release Gate

Critical releases should validate:

```text
Backend build
Backend tests
Frontend build
Frontend tests
Integrated E2E
```

A successful backend pipeline alone does not prove the full product release.

---

# 49. Shared Business Rules

Business rules belong to backend/domain.

Do not duplicate authoritative rules into a shared frontend library.

Frontend may duplicate lightweight validation for UX.

Backend remains authoritative.

---

# 50. Shared Constants

Avoid manual duplication of unstable contract values when generation is practical.

Examples:

```text
Enums
API DTOs
error-code catalogs
```

may be generated or synchronized from backend contract.

However:

```text
product copy
UI labels
theme values
```

belong to frontend.

---

# 51. Permission Keys

Permission keys are backend security contracts.

Frontend may reference them for UX.

Preferred strategy:

```text
backend exposes/generated stable permission keys
```

or frontend keeps a small synchronized catalog.

Do not allow frontend-defined permission names to become security authority.

---

# 52. Feature Keys

Feature keys originate from product/backend entitlement catalog.

Frontend consumes them.

They should remain stable across repositories.

---

# 53. ADR Ownership

All cross-cutting architecture ADRs live in:

```text
shinera-product
```

Examples:

```text
Multi-Tenancy
Authentication
Authorization
Appointment Concurrency
Date/Time
Subscription
Identity
Repository Separation
```

---

# 54. Repository-Specific ADRs

A deeply implementation-specific decision may live in its repository if it does not affect the broader product architecture.

Example:

```text
Angular state-management library choice
```

or:

```text
backend test-container implementation
```

if truly local.

Cross-cutting decisions remain in product repo.

---

# 55. Documentation Hierarchy

Canonical hierarchy:

```text
Product Requirement
↓
Product Backlog
↓
Architecture Overview / ADR
↓
Repository Implementation
↓
Tests
```

Implementation may reveal that documentation needs revision.

Changes must be reconciled rather than silently diverging.

---

# 56. Source of Truth for Current Code

Repository implementation is authoritative for:

```text
what currently exists
```

Product documents are authoritative for:

```text
what is approved/intended
```

If they conflict:

```text
identify conflict
↓
determine whether code or requirement should change
↓
update the correct source
```

---

# 57. Development Status

`DEVELOPMENT-STATUS.md` is cross-repository.

It records:

```text
Backend status
Frontend status
Integration status
Testing status
```

for product Stories.

It does not replace Git history.

---

# 58. Product Backlog

`SHINERA-PRODUCT-BACKLOG.md` remains cross-repository.

Do not create independent competing product backlogs in backend and frontend repositories.

Repository issues may break a Story into implementation tasks.

---

# 59. Repository Issue Backlogs

Backend/frontend repositories may track:

```text
technical debt
bugs
implementation subtasks
repository-specific work
```

but these should reference product Story IDs when tied to product functionality.

---

# 60. Branch Strategy

Each repository may use:

```text
main
+
short-lived feature branches
```

Exact Git Flow is intentionally simple.

Avoid maintaining long-lived environment branches unless CI/deployment requires them.

---

# 61. Feature Branch Naming

Recommended:

```text
feature/SHN-132-create-appointment
fix/SHN-XYZ-...
```

Exact convention is optional, but Story traceability is valuable.

---

# 62. Pull Request Review

PRs are reviewed inside the repository they modify.

Cross-stack Story completion may require:

```text
backend PR merged
frontend PR merged
integration verified
```

before product Story reaches Done.

---

# 63. Definition of Done

`DEFINITION-OF-DONE.md` applies across repositories.

Backend-only Story:

```text
Frontend = N/A
```

Frontend-only Story:

```text
Backend = N/A
```

Cross-stack Story:

```text
both must satisfy applicable gates
```

---

# 64. Repository Permissions

Repository access may differ by contributor/team.

Example future team:

```text
Backend developers
Frontend developers
Product/design
```

Separate repositories allow narrower access if needed.

This is a benefit, not currently a strict requirement.

---

# 65. Secrets and CI Credentials

Each repository owns only credentials required for its pipeline.

Backend secrets:

```text
deployment credentials
container registry
server secrets references
```

Frontend secrets should be limited because browser builds cannot contain private secrets.

Product repo should require almost no runtime secrets.

---

# 66. Docker

If backend uses Docker:

```text
Dockerfile
```

belongs in `shinera-backend`.

If frontend uses Docker/nginx packaging:

```text
Dockerfile
```

belongs in `shinera-frontend`.

---

# 67. Docker Compose

A local development composition file involving both applications may live in:

```text
dedicated development/orchestration location
```

Possible choices:

```text
shinera-product/dev
or
a future shinera-devops repository
```

Do not duplicate conflicting compose files across repositories without purpose.

For MVP, a simple documented local setup is sufficient.

---

# 68. Future Infrastructure Repository

If deployment infrastructure grows substantially, Shinera may later add:

```text
shinera-infrastructure
```

or:

```text
shinera-devops
```

for:

```text
Terraform
Kubernetes
environment manifests
shared deployment orchestration
```

This is not required now.

---

# 69. Why DevOps Repository Is Deferred

Current project does not need a fourth repository merely for theoretical future infrastructure.

Add one when infrastructure code becomes meaningful enough to deserve independent ownership.

---

# 70. Shared Package Repository

Do not create a shared internal package repository merely to avoid a small amount of duplication.

A shared package is justified when code is:

```text
genuinely reusable
versionable
stable
used by multiple applications
```

not simply because two repositories exist.

---

# 71. Backend NuGet Packages

Reusable backend infrastructure such as a future stable library may become its own package/repository only when justified.

Do not prematurely extract core Shinera domain logic into generic packages.

---

# 72. Frontend Shared Packages

Likewise, Angular shared components should remain within Shinera frontend unless multiple separate applications truly require them.

Avoid premature design-system package extraction.

---

# 73. Contract Package Is Rejected

Do not create a manually maintained:

```text
shinera-contracts
```

repository containing duplicated C# and TypeScript DTOs.

OpenAPI/API generation is the preferred cross-stack contract mechanism.

---

# 74. Product Assets

Product-wide non-runtime design references may live in:

```text
shinera-product
```

if useful.

Runtime frontend assets such as:

```text
logos
SVG
images
fonts references
```

used by the Angular app belong in frontend.

---

# 75. Database Ownership

The backend owns the database schema.

No other repository directly modifies production database schema.

Frontend never connects directly to the database.

Product documentation may describe the model but is not executable schema authority.

---

# 76. API Ownership

Backend owns:

```text
API implementation
OpenAPI document
authentication endpoints
authorization enforcement
error contracts
```

Frontend consumes the API.

---

# 77. UI Ownership

Frontend owns:

```text
visual layout
interaction
responsive design
RTL
dark/light theme
client-side UX
```

Backend should not embed business UI.

---

# 78. Business Rule Ownership

Backend/domain owns authoritative business rules.

Frontend may mirror simple validation.

Example:

```text
Service duration required
```

may be validated in UI.

But rules such as:

```text
appointment overlap
subscription limit
tenant ownership
permission scope
```

are backend-authoritative.

---

# 79. Product Decision Ownership

Product repo owns decisions such as:

```text
Does VIP enable Force Appointment?
Which Plan contains Feature X?
What is MVP?
What states does Appointment support?
```

Implementation repositories should not silently invent these decisions.

---

# 80. Release Notes

Product-level release notes may live in:

```text
shinera-product
```

if maintained centrally.

Repository release notes may focus on technical changes.

Do not require duplicate identical release notes everywhere.

---

# 81. Tags

Repository tags represent repository artifacts.

Example:

```text
shinera-backend v1.0.0
shinera-frontend v1.0.0
```

They do not need to be created at exactly the same commit date.

---

# 82. Product Release Marker

A product release such as:

```text
Shinera MVP RC1
```

may reference exact backend/frontend versions or commits.

Example:

```text
Backend: abc123 / v0.9.0
Frontend: def456 / v0.11.0
```

This creates reproducible product releases.

---

# 83. Rollback

Independent repositories enable independent rollback.

Example:

```text
Frontend regression
→ rollback frontend only
```

if backend remains compatible.

Database-destructive backend migrations complicate rollback and therefore require extra care.

---

# 84. Feature Rollout

A backend capability may be deployed before frontend exposure.

Example:

```text
backend endpoint deployed
↓
frontend feature deployed later
```

This is acceptable when the endpoint is secure and backward-compatible.

---

# 85. Dead Code Across Repositories

Do not keep old backend endpoints indefinitely solely because frontend once used them.

When deprecating:

```text
verify frontend no longer depends on it
↓
remove in planned cleanup
```

For external consumers, formal deprecation rules may be needed later.

---

# 86. Local Developer Workflow

Typical cross-stack task:

```text
Read Story / ADR
↓
Open backend repo
↓
Implement API/domain
↓
Run backend tests
↓
Open frontend repo
↓
Integrate API/UI
↓
Run frontend build/tests
↓
Run E2E
↓
Update Development Status
```

One developer may work across both repositories.

Repository separation does not imply team separation.

---

# 87. ChatGPT / Work Workflow

Product and architecture planning references:

```text
shinera-product
```

Implementation tasks should inspect the actual target repository before modifying code.

For cross-stack work:

```text
backend and frontend
```

must both be inspected.

Agents must not assume repository state from documentation alone.

---

# 88. Agent Task Format

Recommended task instruction:

```text
Story: SHN-XXX
Repository: shinera-backend / shinera-frontend / both

References:
- PRD
- Product Backlog
- Project Instructions
- Relevant ADRs

Instruction:
Inspect current implementation first.
Implement approved scope only.
Build and test.
Report repository-specific results.
```

---

# 89. Repository Sync

Because repositories may temporarily lag local work, GitHub must not be treated as fully authoritative until active implementation has been pushed.

After initial synchronization:

```text
GitHub repository
```

becomes the primary current implementation reference.

`DEVELOPMENT-STATUS.md` must be reconciled after major syncs.

---

# 90. Repository Bootstrap

Initial bootstrap sequence:

```text
1. Push active backend into shinera-backend
2. Push active frontend into shinera-frontend
3. Commit product docs into shinera-product
4. Verify builds
5. Verify tests
6. Reconcile Development Status
```

---

# 91. Product Repo Initial Documents

Initial canonical set:

```text
SHINERA-PRD.md
SHINERA-PRODUCT-BACKLOG.md
PROJECT-INSTRUCTIONS.md
DEVELOPMENT-STATUS.md
DEFINITION-OF-DONE.md
ARCHITECTURE-OVERVIEW.md
ADR-001...
ADR-002...
ADR-003...
ADR-004...
ADR-005...
ADR-006...
ADR-007...
ADR-008...
```

---

# 92. Dependency Direction Across Repositories

Conceptually:

```text
Product Decisions
      ↓
Backend API
      ↓
Frontend Integration
```

But feedback may flow upward:

```text
Implementation discovery
      ↓
Architecture/Product review
      ↓
Document update
```

This is not a package dependency graph.

---

# 93. No Git Submodules by Default

Do not use Git submodules to embed:

```text
product repo into backend
frontend into backend
backend into frontend
```

Submodules create synchronization and tooling friction without meaningful benefit here.

Repositories should remain independently cloned.

---

# 94. No Copy Scripts for Product Docs

Do not create scripts that automatically copy PRD/ADR files into every repository.

Use links/references instead.

One canonical copy is safer.

---

# 95. README Minimalism

Repository README should include enough to:

```text
understand repository purpose
run locally
build/test
find product documentation
```

README should not become a duplicate architecture handbook.

---

# 96. Backend README

Recommended sections:

```text
Shinera Backend
Prerequisites
Local Setup
Database
Build
Tests
API/Scalar
Environment Configuration
Product Docs Link
```

---

# 97. Frontend README

Recommended sections:

```text
Shinera Frontend
Prerequisites
Install
Run
Build
Tests
Playwright
Environment Configuration
Product Docs Link
```

---

# 98. Product README

Recommended sections:

```text
Shinera
Product Overview
Repository Map
Documentation Index
Current Status
Development Workflow
Links to backend/frontend
```

---

# 99. Cross-Repository Links

Documentation should use stable repository links where useful.

Avoid links to temporary branches.

Prefer:

```text
main
or
stable file paths
```

unless documentation is version-specific.

---

# 100. Product Documentation Versioning

Architecture decisions evolve through Git history and ADR status.

Do not maintain:

```text
ADR-final-final-v3.md
```

Use:

```text
ADR status
Git commits
Superseded-by references
```

for evolution.

---

# 101. ADR Supersession

If a decision changes:

```text
old ADR
→ Status: Superseded

new ADR
→ references old ADR
```

Do not silently rewrite architectural history after implementation has depended on it.

Minor clarifications may update existing ADR when meaning does not materially change.

---

# 102. Release Branches

Long-lived release branches are not required initially.

Use:

```text
main
+
tags/releases
```

unless production operations later justify release branches.

---

# 103. Hotfix

A hotfix belongs to the affected repository.

Example:

```text
backend auth bug
→ backend hotfix
```

Cross-stack hotfix only touches both repositories when both actually need changes.

---

# 104. CI Trigger Scope

Backend CI triggers on backend repository changes.

Frontend CI triggers on frontend changes.

Product doc changes should not rebuild backend/frontend automatically unless a deliberate integration workflow requires it.

This is a major benefit of repository separation.

---

# 105. Dependency Bot / Security Updates

Backend dependency automation handles:

```text
NuGet
.NET packages
```

Frontend handles:

```text
npm
Angular packages
```

This avoids mixed dependency noise.

---

# 106. Code Ownership

Future `CODEOWNERS` may independently reflect:

```text
backend reviewers
frontend reviewers
architecture/product reviewers
```

This is optional until team size requires it.

---

# 107. Deployment Responsibility

Backend deployment owns:

```text
API runtime
database migration coordination
backend health
```

Frontend deployment owns:

```text
static/client runtime
frontend availability
asset delivery
```

Product repo has no runtime SLO.

---

# 108. Health Checks

Backend provides health endpoints.

Frontend availability monitoring is deployment/platform-specific.

Repository separation makes these operational concerns independent.

---

# 109. Observability

Backend observability:

```text
API requests
traces
database
exceptions
health
```

Frontend observability:

```text
client errors
UX performance
browser failures
```

They may feed the same monitoring platform without being in one repository.

---

# 110. Security Boundary

Repository separation itself is not a security boundary.

Frontend remains untrusted client software.

Backend must enforce:

```text
authentication
authorization
tenant isolation
feature entitlement
business rules
```

regardless of frontend repository ownership.

---

# 111. Deployment Topology Independence

Separate repositories do not force:

```text
separate servers
separate domains
containers
Kubernetes
```

They only allow independent artifacts/lifecycles.

Deployment topology may remain simple.

---

# 112. Initial Deployment Recommendation

Conceptually:

```text
Frontend
→ static web hosting / web server

Backend
→ ASP.NET Core service/container

Database
→ relational database
```

Exact infrastructure is outside this ADR.

---

# 113. Same-Origin Future

If frontend and backend are later served under one origin:

```text
https://app.shinera...
```

the repositories may still remain separate.

Repository structure and network topology are independent decisions.

---

# 114. BFF Future

If ADR-002 later evolves to BFF:

```text
frontend repository
```

may remain separate from:

```text
backend/BFF implementation
```

or a dedicated BFF may be introduced.

This does not automatically require a monorepo.

---

# 115. Mobile App Future

A future mobile app should likely become:

```text
shinera-mobile
```

and consume the same backend APIs.

This repository strategy scales naturally to additional clients.

---

# 116. Customer App Future

If customer-facing experience becomes a separate application:

```text
shinera-customer
```

may become a separate repository if operationally justified.

For MVP, customer-facing UI may remain inside `shinera-frontend`.

---

# 117. Admin Platform Future

A future internal Shinera platform admin UI may become:

```text
shinera-admin
```

only if deployment/team boundaries justify it.

Do not split prematurely.

---

# 118. Rejected Alternatives

Rejected for current architecture:

```text
Single monorepo
Duplicate PRD in every repository
Duplicate Backlog in every repository
Git submodules between repositories
Manual shared DTO repository
Coupled backend/frontend version numbers
Frontend database access
Per-Plan deployments
One CI pipeline rebuilding everything for every documentation change
```

---

# 119. Consequences

## Positive

This decision provides:

```text
Clear ownership
Independent CI
Independent deployment
Independent rollback
Smaller repository scope
Cleaner technology boundaries
Simple product documentation authority
Future multi-client extensibility
```

## Negative

It introduces:

```text
Cross-repository coordination
Potential API contract drift
Multiple PRs for one Story
Need for release compatibility discipline
Need for central product tracking
```

These costs are accepted.

---

# 120. Mitigations for Cross-Repository Cost

Use:

```text
Stable Story IDs
OpenAPI/client generation
GitHub Project aggregation
Product repo documentation
Development Status
Cross-stack E2E
Backward-compatible API changes
```

This provides coordination without collapsing repositories.

---

# 121. Revisit Conditions

Revisit repository strategy if:

```text
most changes require atomic cross-repo commits
contract synchronization becomes a major bottleneck
shared tooling becomes dominant
CI/dependency management becomes harder because of separation
team structure changes materially
```

Do not merge repositories solely for convenience during a temporary development phase.

---

# 122. Definition of Done Integration

Cross-stack Story cannot be marked Done merely because one repository is complete.

Example:

```text
Backend Done
Frontend In Progress
→ Overall Story = In Progress
```

Done requires all applicable repository work plus integration verification.

---

# 123. Final Decision Summary

Shinera uses:

```text
shinera-product
shinera-backend
shinera-frontend
```

as three independent Git repositories.

Ownership:

```text
shinera-product
→ What / Why / Architecture / Status / Governance

shinera-backend
→ Domain / Data / API / Security / Backend Tests

shinera-frontend
→ UX / Angular / API Consumption / E2E
```

The repositories are independently:

```text
versioned
built
tested
deployed
rolled back
```

while remaining one coordinated product through:

```text
Story IDs
Product Backlog
Architecture Decisions
API Contracts
Development Status
Integrated E2E
```

Product documents have one canonical home and are not duplicated across backend/frontend repositories.

The HTTP/OpenAPI contract is the primary backend/frontend integration boundary.

Shinera deliberately chooses repository separation without introducing microservices or unnecessary organizational complexity.
