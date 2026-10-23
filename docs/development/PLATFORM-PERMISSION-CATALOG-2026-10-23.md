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
4. **Do not enable for a production security rollout until Permission CRUD, grant-mutation routes, and privilege self-escalation paths are protected and tested.** The current feature flag gates only five Role mutations; broader administration endpoints remain on the coarse SystemAdministrator policy. After those dependencies, grant migration verification, HTTP 401/403/revocation testing, and a rollback plan, enable `Identity:PlatformPermissions:EnforceRoleEndpoints=true` and restart the application. The default false state preserves the old protected paths.
5. After rollout, test dynamic revocation, cross-tenant assignments, backend audit and frontend effective permission updates. Do not re-seed a revoked grant.

**Important:** Current CI's optional Development grants are deliberately repeatable for disposable installations; leave this flag off for revocation testing and production. Catalog definition reconciliation itself must never restore missing grant rows.

## Verification and limitations

- SQL Server test on an isolated migrated SQL Server database verifies definition idempotence, default deny, explicit Development grant, and that removing a RolePermission persists across catalog-only re-seeding, including preservation of deactivated catalog keys.
- CI opt-in workflow sets both flags only for a disposable Development application and validates live System login, Role creation and cross-scope denial.
- Full production migration, explicit Permission CRUD boundaries, tamper-resistant operation auditing, all Role HTTP 401/403 actions, privileged self-escalation tests and raw SQL/bulk write audits remain open.
- **No Production activation and no complete ADR-003 claim.** The reviewed backend slice can merge with the enforcement switch left OFF by default; no production-wide privilege boundary is claimed.

## Evidence ledger

| Gate | Evidence | State |
|---|---|---|
| Backend Release Build + xUnit | [CI 39872147609](https://github.com/sholehamed/Shinera-backend/actions/runs/39872147609) | **21/21 passed**, Release Build successful |
| SQL Server migration and permissions | [CI 39872147628](https://github.com/sholehamed/Shinera-backend/actions/runs/39872147628) | **14/14 passed**, real SQL Server migrations |
| Disposable real HTTP opt-in Role creation | [CI 39872147628](https://github.com/sholehamed/Shinera-backend/actions/runs/39872147628) | **Passed**, System login, protected read, authorized Role POST and foreign-scope 403 |
| Production migration and enforcement | Not performed | Explicit later rollout |

## Final pre-merge security review — 2026-10-23

- Inspected the five-file backend diff (catalog, seeding, Role route metadata, SQL Server regression, CI opt-in configuration) and confirmed clean GitHub mergeability of [PR #13](https://github.com/sholehamed/Shinera-backend/pull/13) at `9f160ea37925484e2bce1f0f44682f7d39c8c4ba`.
- [Final PR-head Backend CI](https://github.com/sholehamed/Shinera-backend/actions/runs/39872147609): **21/21 xUnit + Release Build succeeded**. [Final PR-head SQL Server CI](https://github.com/sholehamed/Shinera-backend/actions/runs/39872147628): **14/14 integration tests + disposable Development API authentication/Role-create/cross-scope rejection passed**.
- SeedDefinitionsAsync never automatically restores RolePermission grants or reactivates inactive definitions. Production never invokes GrantDevelopmentSuperAdminAsync through the startup code. Both granular Role endpoint checks and Development grant bootstrap are OFF by default.
- **Security scope limitation:** the SQL Server HTTP smoke demonstrates positive operation with the opt-in granted, not a full negative HTTP matrix for revoked `ROLES.CREATE`. Permission CRUD and other grant-mutation paths still use the broader SystemAdministrator policy; a privileged operator could potentially modify permission definitions/grants outside the newly gated Role endpoints. Until these paths are locked down, **do not use the feature flag in Production as proof of granular least privilege**.
- BootstrapDevelopmentGrants deliberately restores any missing Development SuperAdmin grants on startup if left enabled. It must remain OFF for revocation tests and non-disposable deployments. Additional production migration, credential and durable authorization audit work remains in [Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11).
