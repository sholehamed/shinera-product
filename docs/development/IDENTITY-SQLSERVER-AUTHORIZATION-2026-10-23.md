# Shinera — SQL Server Identity & ADR-003 Authorization — 2026-10-23

**State:** First SQL Server/tenant-permission slice **merged** to backend main; complete ADR-003 still open  
**Linked work:** [Backend PR #7](https://github.com/sholehamed/Shinera-backend/pull/7) · [P0 Issue #4](https://github.com/sholehamed/Shinera-backend/issues/4) · [P0 Issue #5](https://github.com/sholehamed/Shinera-backend/issues/5)  
**Architecture decisions:** [ADR-001](../architecture/decisions/ADR-001-MULTI-TENANCY-STRATEGY.md), [ADR-003](../architecture/decisions/ADR-003-AUTHORIZATION-AND-PERMISSION-MODEL.md)

## Working branches

- Backend: `security/sqlserver-authorization-regression-20261023`
- Product: `docs/identity-authorization-sqlserver-20261023`

## Initial code review: source-backed findings

- Current `RefreshGrantValidator` checks the authenticated account, SecurityStamp, roles/permissions, tenant identity and tenant active status on refresh; earlier database mutation regression tests ran on SQLite, **not SQL Server**.
- Current `ActiveSystemAdministratorHandler` reads current user/role from DB, so unexpired JWT role claims alone cannot authorize a revoked platform administrator.
- `IdentityDbContext` applies named EF Core `tenant` query filters to `IMustHaveTenant` entities based on `TenantContext.ReadScope`; explicit `IgnoreQueryFilters(["tenant"])` bypasses this barrier for trusted internal operations.
- `PermissionMiddleware` is intentionally disabled. Its current logic skips endpoints with `IAuthorizeData`, and `DbPermissionChecker` requires a `tenant_id` claim. `RoutePermissionResolver` synthesizes permission identifiers from a route/method rather than stable `resource.action` keys. Enabling this middleware in its present form would not implement ADR-003.
- ADR-003 requires explicit **Resource + Action + Scope**, separate platform and Tenant authorization, default deny, and frontend visibility as UX only. Scope semantics include Tenant, Branch, Own and reserved Child; Subscription feature gating is separate.

## Implemented in Backend PR #7

- New `Shinera.Identity.SqlServerTests` project using actual SQL Server provider.
- GitHub Actions PR workflow starts isolated SQL Server 2022 container and runs the tests.
- Every test uses a unique random test database, runs actual EF Core migrations, asserts the security behavior, and destroys the database.
- Revoked System administrator role prevents both refresh and subsequent privileged authorization using pre-existing token claims.
- Disabled System account prevents refresh.
- Tenant A role read scope does not expose Tenant B role; explicit privileged query-filter bypass is covered separately.
- Suspended Tenant A prevents subsequent refresh with unchanged claims.
- CI stores TRX logs; tests never target production or a shared database.

## First explicit Resource + Action authorization building block

- Added `TenantPermissionRequirement`, `TenantPermissionHandler`, and `RequireTenantPermission(resource, action)` extension in Backend PR #7.
- Stable permission keys are validated and normalized as `RESOURCE.ACTION`. Requests without a verified authenticated Tenant account, active home-tenant, active user or matching explicit Permission fail closed.
- Effective grants are resolved from active direct User permissions, direct Role grants, or Group → Role grants, restricted to the subject's own Tenant. Revoked/inactive grants and cross-Tenant claims do not grant permission.
- This is **Tenant scope only** and intentionally **not yet attached to existing System admin endpoint groups**. Their established coarse SystemAdministrator policy remains in effect. It must not be mistaken for complete ADR-003 scope handling.
- Added a third real SQL Server integration test for exact resource/action match, role grant, role revocation, direct grant, wrong Tenant, forged System context and inactive Tenant.
- **Final review fix:** explicit user-level denial (`IsGranted=false`) overrides inherited Role/Group grants, preserving the legacy permission checker behavior. Group and GroupRole rows must also match the current Tenant. A fourth SQL Server test covers both cases.

## Next implementation slices (not completed)

1. Apply the new explicit `RequireTenantPermission(resource, action)` to real business endpoints as they are implemented; add actual HTTP 401/403 tests, not only requirement-handler integration tests.
2. Build separate System permission policies and tenant Branch/Own scope evaluators with defined membership and ownership. Preserve the platform/tenant boundary.
3. Resolve how the legacy master-Tenant record stores platform role definitions without treating the platform administrator itself as a Tenant.
4. Verify token rotation/reuse semantics, production signing keys, OpenIddict client bootstrap and Subscription gating separately.

## Verification ledger

| Check | Status |
|---|---|
| GitHub Actions Backend SQL Server integration tests on PR #7 | **4/4 passed** — [final head run 39857222037](https://github.com/sholehamed/Shinera-backend/actions/runs/39857222037) |
| Backend CI at final PR head | **21/21 xUnit passed; Build successful** — [run 39857222206](https://github.com/sholehamed/Shinera-backend/actions/runs/39857222206) |
| Resource/Action/Scope authorization enforcement | Tenant-scope `Resource.Action` handler implemented and database-tested, **not yet applied to product endpoints**; System/Branch/Own remain open |

## Documentation rule

When this PR is updated/merged, write exact code commit SHA, test counts and workflow URLs here and append a dated record to `docs/development/ENGINEERING-LOG-2026-10-23.md`. Never describe tests as successful before their GitHub Actions runs complete.

## Final review and backend merge

- [Backend PR #7](https://github.com/sholehamed/Shinera-backend/pull/7) reviewed against final head `9170a08f7a0b73c19f99904ef3cdd18f12014f9c`, verified no merge conflict and both checks successful.
- Corrected two authorization gaps before merge: explicit denial precedence over Role/Group and cross-tenant Group/GroupRole mismatch. Added fourth SQL Server regression.
- [SQL Server CI 39857222037](https://github.com/sholehamed/Shinera-backend/actions/runs/39857222037): **4/4 passed**. [Backend CI 39857222206](https://github.com/sholehamed/Shinera-backend/actions/runs/39857222206): **21/21 xUnit passed**, Release Build successful.
- Backend PR #7 **squash merged** into `main` at [ad1bdad1](https://github.com/sholehamed/Shinera-backend/commit/ad1bdad1d73423b75da3c662b696579a9f1e5208). The new handler is an available authorization primitive, not a blanket activation of unfinished permission policies.
