# Shinera — Platform Permission Catalog and staged enforcement

**Date:** 2026-10-23  
**State:** Draft — not merged and NOT enabled in Production  
**Backend implementation:** [Draft PR #13](https://github.com/sholehamed/Shinera-backend/pull/13)  
**Architecture:** [ADR-003](../architecture/decisions/ADR-003-AUTHORIZATION-AND-PERMISSION-MODEL.md)  
**Related:** [Backend Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11), [Tenant write isolation Issue #9](https://github.com/sholehamed/Shinera-backend/issues/9)

## Why stage the rollout

Existing System endpoint groups use `SystemAdministrator` (current DB-validated System account and master-scoped SuperAdmin role), whereas `RequireSystemPermission(resource, action)` is a separate grant-based ADR-003 requirement. Applying fine-grained requirements to live endpoints without first migrating existing persisted grants would revoke platform administrators on deployment. Conversely, implicit SuperAdmin bypass would invalidate explicit Resource.Action checks.

The approved architecture distinguishes System users (`User.TenantId = null`) from the legacy Master Tenant id used only for storage of platform Role and Permission assignments. No System user is given a tenant home as part of this migration.

## Static catalog, keys and endpoint mapping

Stable database keys are upper-case. The endpoint helper takes validated lower-case resource and action components:

| Resource | Action | Stored Code | Rollout |
|---|---|---|---|
| roles | create | ROLES.CREATE | Optional, Role POST |
| roles | update | ROLES.UPDATE | Optional, Role PUT |
| roles | delete | ROLES.DELETE | Optional, Role DELETE |
| roles | change-state | ROLES.CHANGE-STATE | Optional, Role PATCH |
| roles | assign-permissions | ROLES.ASSIGN-PERMISSIONS | Optional, Role permission replacement PUT |
| permissions | create | PERMISSIONS.CREATE | Catalog only; not yet enforced |
| permissions | update | PERMISSIONS.UPDATE | Catalog only; not yet enforced |
| permissions | delete | PERMISSIONS.DELETE | Catalog only; not yet enforced |
| permissions | change-state | PERMISSIONS.CHANGE-STATE | Catalog only; not yet enforced |

`PlatformPermissionCatalog` owns these keys and `PlatformPermissionCatalogSeeder.SeedDefinitionsAsync` adds missing definitions at startup without granting privileges, reactivating disabled existing records or overwriting existing definitions. The current `Permission` entity is a shared global definition; `RolePermission` retains the master storage TenantId for platform assignments.

## Feature flags and safe operator sequence

The new flags are **OFF by default**:

- `Identity:PlatformPermissions:EnforceRoleEndpoints`: opt-in adds fine-grained requirements on the five mutating Role routes, **in addition to** `SystemAdministrator`.
- `Identity:PlatformPermissions:BootstrapDevelopmentGrants`: when running in **Development only**, deliberately assigns active catalog entries to exactly one active master-scoped SuperAdmin role. Used only by disposable CI/test environments. Never set as a Production bootstrap mechanism.

**Production rollout runbook (not executed yet):**

1. Verify all current platform administrators' master-scope role assignments, existing permissions and grant revocation history; back up the database and inventory active integrations.
2. Review and approve a **one-time** master-scoped RolePermission grant migration with actor/operator audit. Do not repeat grants on every application startup.
3. Verify current authorized admin gets expected allowed actions; confirm non-granted users and inactive accounts fail with 403 and anonymous users with 401.
4. Enable `Identity:PlatformPermissions:EnforceRoleEndpoints=true` after migration verification and a rollback plan. The default false state preserves the old protected paths.
5. After rollout, test dynamic revocation, cross-tenant assignments, backend audit and frontend effective permission updates. Do not re-seed a revoked grant.

**Important:** Current CI's optional Development grants are deliberately repeatable for disposable installations; leave this flag off for revocation testing and production. Catalog definition reconciliation itself must never restore missing grant rows.

## Verification and limitations

- SQL Server test on an isolated migrated SQL Server database verifies definition idempotence, default deny, explicit Development grant, and that removing a RolePermission persists across catalog-only re-seeding, including preservation of deactivated catalog keys.
- CI opt-in workflow sets both flags only for a disposable Development application and validates live System login, Role creation and cross-scope denial.
- Full production migration, explicit Permission CRUD boundaries, tamper-resistant operation auditing, all Role HTTP 401/403 actions, privileged self-escalation tests and raw SQL/bulk write audits remain open.
- **No Production activation, no merge, and no complete ADR-003 claim in this phase's initial record.**

## Evidence ledger

| Gate | Evidence | State |
|---|---|---|
| Backend Release Build + xUnit | [CI 39872147609](https://github.com/sholehamed/Shinera-backend/actions/runs/39872147609) | **21/21 passed**, Release Build successful |
| SQL Server migration and permissions | [CI 39872147628](https://github.com/sholehamed/Shinera-backend/actions/runs/39872147628) | **14/14 passed**, real SQL Server migrations |
| Disposable real HTTP opt-in Role creation | [CI 39872147628](https://github.com/sholehamed/Shinera-backend/actions/runs/39872147628) | **Passed**, System login, protected read, authorized Role POST and foreign-scope 403 |
| Production migration and enforcement | Not performed | Explicit later rollout |
