# Shinera Engineering Log — 2026-10-23 — System Authorization Follow-up

## New scoped authorization phase

- Created Backend branch `security/system-resource-permissions-20261023`, [Draft PR #8](https://github.com/sholehamed/Shinera-backend/pull/8), from the latest main after SQL Server regression baseline.
- Created Product branch `docs/system-resource-permissions-20261023` to record architecture, scope, implementation and CI evidence.
- Reviewed accepted [ADR-003](../architecture/decisions/ADR-003-AUTHORIZATION-AND-PERMISSION-MODEL.md) and current Identity code.
- Added platform-specific `SystemPermissionRequirement` and `SystemPermissionHandler`, explicit `RequireSystemPermission` Minimal API extension, database-backed grants restricted to the fixed legacy Master Tenant permission storage scope. Platform System User retains nullable TenantId.
- Added two SQL Server regression tests for platform direct/role/group grants, explicit denial, cross-Tenant scope and tenant-account rejection.
- Identified Branch/Own dependencies not yet present in current backend; deliberately deferred speculative scope handlers and retained existing SystemAdministrator protection.
- This log records ongoing Draft work: no PR merged and no CI outcome is claimed until confirmed.

[Detailed system permission implementation and rollout plan](SYSTEM-RESOURCE-PERMISSIONS-2026-10-23.md).

- First [Backend CI 39858625068](https://github.com/sholehamed/Shinera-backend/actions/runs/39858625068) **21/21 xUnit passed** and first [SQL Server CI 39858625169](https://github.com/sholehamed/Shinera-backend/actions/runs/39858625169) **6/6 integration tests passed**. A subsequent change expanded the foreign-Tenant grant regression; its final head CI is still being verified.
- During source review discovered disabled write scope and TenantId immutability in `TenantSaveChangesInterceptor`; recorded [P0 security issue #9](https://github.com/sholehamed/Shinera-backend/issues/9) with explicit platform-write bypass design and SQL Server acceptance tests.
