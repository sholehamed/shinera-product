# Shinera Engineering Log — 2026-10-23 — System Authorization Follow-up

## New scoped authorization phase

- Created Backend branch `security/system-resource-permissions-20261023`, [Draft PR #8](https://github.com/sholehamed/Shinera-backend/pull/8), from the latest main after SQL Server regression baseline.
- Created Product branch `docs/system-resource-permissions-20261023` to record architecture, scope, implementation and CI evidence.
- Reviewed accepted [ADR-003](../architecture/decisions/ADR-003-AUTHORIZATION-AND-PERMISSION-MODEL.md) and current Identity code.
- Added platform-specific `SystemPermissionRequirement` and `SystemPermissionHandler`, explicit `RequireSystemPermission` Minimal API extension, database-backed grants restricted to the fixed legacy Master Tenant permission storage scope. Platform System User retains nullable TenantId.
- Added two SQL Server regression tests for platform direct/role/group grants, explicit denial, cross-Tenant scope and tenant-account rejection.
- Identified Branch/Own dependencies not yet present in current backend; deliberately deferred speculative scope handlers and retained existing SystemAdministrator protection.
- This log records the scoped code work and its final successful CI and merge; broader ADR-003 rollout remains outstanding.

[Detailed system permission implementation and rollout plan](SYSTEM-RESOURCE-PERMISSIONS-2026-10-23.md).

- First [Backend CI 39858625068](https://github.com/sholehamed/Shinera-backend/actions/runs/39858625068) **21/21 xUnit passed** and first [SQL Server CI 39858625169](https://github.com/sholehamed/Shinera-backend/actions/runs/39858625169) **6/6 integration tests passed**. Final [Backend CI 39859081756](https://github.com/sholehamed/Shinera-backend/actions/runs/39859081756) **21/21 passed**, and [SQL Server CI 39859082403](https://github.com/sholehamed/Shinera-backend/actions/runs/39859082403) **6/6 passed** on latest code including the expanded foreign-Tenant role regression.
- During source review discovered disabled write scope and TenantId immutability in `TenantSaveChangesInterceptor`; recorded [P0 security issue #9](https://github.com/sholehamed/Shinera-backend/issues/9) with explicit platform-write bypass design and SQL Server acceptance tests.

## Final review and merge

- [Backend PR #8](https://github.com/sholehamed/Shinera-backend/pull/8) **merged** to main via [`664b8bb9`](https://github.com/sholehamed/Shinera-backend/commit/664b8bb95a3926b292d5ae4703505627a95741c5). Last reviewed head was `cd247db8990c456ad63d43127b864e9e074d3d06`.
- [Backend CI 39859081756](https://github.com/sholehamed/Shinera-backend/actions/runs/39859081756): **21/21 xUnit + Release Build**; [SQL Server 39859082403](https://github.com/sholehamed/Shinera-backend/actions/runs/39859082403): **6/6** on final PR head.
- Permission handler remains **opt-in**. Current privileged System API endpoints continue using `SystemAdministrator`. Activate granular permissions only with a catalog, grant migration, compatibility plan and HTTP 401/403 tests.
- P0 TenantSaveChangesInterceptor write-isolation flaw is open as [Issue #9](https://github.com/sholehamed/Shinera-backend/issues/9).
