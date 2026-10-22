# Shinera — Test Strategy

**Active baseline:** 2026-10-22

## Ownership
- **Backend repository:** xUnit unit/application/integration tests for .NET 10, EF Core, OpenIddict and SQL Server.
- **Frontend repository:** Angular 22 Vitest/component tests and Playwright browser/API E2E with real ASP.NET Core and isolated SQL Server.
- **Product repository:** authoritative test strategy, coverage inventory, engineering logs and ADR/PRD acceptance criteria.

## Test tiers
1. Unit: pure domain rules, middleware/context, validation, Angular reactive forms and signals. Run on every PR.
2. Application/integration: WebApplicationFactory + isolated SQL Server; test migrations, EF query filters, Identity protocol and permission policies. **Not implemented yet.**
3. API: assert exact 401/403 and valid response contracts for anonymous, unauthorized and permitted subjects. Run on PR.
4. Browser E2E: verify critical user flows through Angular with the real backend and database. No mock business APIs.

## P0 Identity & System scenarios
- Admin correct login and appropriate root `/system/dashboard` navigation.
- Wrong password, missing/invalid/expired CAPTCHA, inactive account.
- Access token / refresh / expiration / revocation / Logout.
- System account has no TenantId; tenant accounts cannot invoke privileged System operations.
- Unauthorized access fails even through direct API calls, not only hidden UI controls.
- Cross-tenant read/write/delete denial across real seeded tenants and related entities.

**Important:** the initial CI test baseline only proves smoke tests and a few System/Tenant unit behaviors. It is not yet an Identity security certification. See [P0 backend issue](https://github.com/sholehamed/Shinera-backend/issues/2).

## MVP features
Each story must include: valid case, invalid/boundary inputs, authorized and unauthorized caller tests, tenant isolation tests, and (where relevant) business concurrency tests. Appointment booking requires concurrent SQL Server requests, not only frontend tests.

## CI principles
- Never use development/production SQL Server for CI. Use ephemeral container and isolated fixture data.
- Commit dependency lockfiles and use `npm ci` once possible; pin backend source revision used by E2E.
- Keep E2E worker count limited until tests can run independently.
- Store Playwright traces, screenshots, and HTML reports on failure; TRX for xUnit.
- Link evidence to GitHub Actions run URLs.
- Never commit production credentials, secrets or live user data, and review public CI logs/artifacts.
- Required Check gates for main branch merges should include backend/frontend Build+unit and critical full-stack E2E once scope and triggers are stabilized.

## Documentation expectations
Store engineering history in `docs/development/ENGINEERING-LOG-YYYY-MM-DD.md`; update `docs/development/DEVELOPMENT-STATUS.md` and `docs/testing/TEST-COVERAGE.md` per merged milestone.
