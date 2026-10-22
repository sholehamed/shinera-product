# Shinera — System Role/Permission Write Path — 2026-10-23

**State:** Work in progress; PRs Draft and not merged  
**Tracked:** [Backend Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11), [Tenant Write Isolation Issue #9](https://github.com/sholehamed/Shinera-backend/issues/9)  
**Backend:** [PR #12](https://github.com/sholehamed/Shinera-backend/pull/12) (`security/platform-role-permission-writes-20261023`)  
**Product docs:** `docs/platform-role-permission-writes-20261023`

## Source findings

- The earlier merged TenantSaveChangesInterceptor makes normal `IMustHaveTenant` writes fail closed for System users without a trusted operation scope. `TrustedTenantWriteScope.BeginPlatformMasterWrite` is internal and bounded to the fixed master definition/assignment tenant.
- `RoleCreateCommandHandler` previously mapped the wrong command type and did not fill Role.TenantId for System context.
- `UpdateRolePermissionsCommandHandler` used empty TenantId on new RolePermission rows.
- The old Role commands ran against EF tenant query filters. System context has an empty `ReadScope`, so a permissioned master Role could not be located without a **temporary** read bypass.
- The existing coarse SystemAdministrator endpoint policy remains in use; fine-grained System `Resource.Action` catalog/migration is not yet deployed. System users must retain `User.TenantId == null`.

## Current implementation

1. `PlatformRoleWriteExecutor` checks authenticated System principal and JWT role, matches the `ITenantContext` subject, verifies live active System user, live SuperAdmin role assignment specifically in the master storage scope, and active master storage tenant. Targets with a non-master Role.TenantId are denied.
2. Only after verification, a transient `tenant.DisableFilter()` **read** scope and infrastructure `TrustedTenantWriteScope.BeginPlatformMasterWrite` **write** scope are opened for the command; both are disposed and operations logged with actor/operation/role identifiers.
3. All legacy System Role mutating endpoints (create, update, delete, change status and replace permissions) are routed through this executor. Existing `SystemAdministrator` endpoint policy stays applied; the code does not grant an arbitrary Tenant write bypass.
4. RoleCreate AutoMapper mapping targets RoleCreateCommand, and a server-owned JSON-ignored `StorageTenantId` is populated from the fixed master storage id. RolePermission creation also receives that verified storage scope and validates the target Role belongs to it.
5. SQL Server tests added for valid scoped Role creation, revoked admin role, foreign Role target, default-deny outside scope, valid RolePermission replacement and retention of existing assignment when an invalid permission is requested.

## Boundaries and follow-up P0 work

- This is the **Role write slice**, not finished catalog-level Permission CRUD, UserRole/GroupRole assignment, complete ADR-003 grant migration, or production System permissions.
- The built-in SystemAdministrator gate currently protects these endpoints. Future fine-grained `RequireSystemPermission("roles", "manage")` requires a backward-compatible catalog + grant bootstrap, otherwise existing admins will be locked out.
- The Role mutating endpoint handlers now catch denied platform-scope operations and return an explicit HTTP **403** instead of a generic 500. Real HTTP testing is added to the SQL Server CI workflow; its final run must pass before merging.
- The SQL Server CI now sends an actual authorized HTTP POST to create a platform Role and expects 201, followed by a forbidden role-permission write expecting 403. Complete PUT/DELETE/PATCH route coverage is still a follow-up. Existing CI runs before this addition do not prove these requests.
- The System user cannot select an arbitrary TenantId, and the master scope remains a legacy storage detail, not a parent tenancy.
- Role writes performed outside this executor must still fail through the interceptor. Bypass via EF bulk or raw SQL is a separate audit item.
- [Issue #9](https://github.com/sholehamed/Shinera-backend/issues/9) remains open until all tenant-write acceptance criteria and bypass reviews are completed.

## CI and verification ledger

| Test | Status | Evidence |
|---|---|---|
| Backend Build + unit tests | **21/21 passed, Build successful** | [Backend CI run 39868009455](https://github.com/sholehamed/Shinera-backend/actions/runs/39868009455) |
| SQL Server migration/role/assignment tests | **12/12 passed** | [SQL Server CI run 39868009445](https://github.com/sholehamed/Shinera-backend/actions/runs/39868009445) |
| API-level Role HTTP smoke | **Passed** — real seeded System login, protected GET, Role POST 201, invalid-scope permission PUT 403 | [SQL Server CI run 39868009445](https://github.com/sholehamed/Shinera-backend/actions/runs/39868009445); other CRUD routes not yet E2E-tested |
| Production migration/role grant bootstrap | Not implemented | Requires separate rollout decision |

## Recorded implementation issues

- Initial Build failed because `System.Text.Json.Serialization.JsonIgnore` in System-prefixed namespaces resolved to a project namespace. Fixed by using `global::System.Text.Json.Serialization.JsonIgnore`, with new CI run.
- Keep all PRs Draft until latest CI and complete security regression review.

## Additional review changes

- Hardened the existing `ActiveSystemAdministratorHandler`: only SuperAdmin assignments whose `UserRole.TenantId` **and** `Role.TenantId` match the fixed Master Tenant scope can authorize an active platform administrator. A foreign Tenant cannot grant System administrator status by creating a role named SuperAdmin.
- Updated original SQLite and SQL Server authorization regression fixtures to use the actual Master definition scope, not a random tenant identifier.
- Role mutation endpoints now return `ForbidHttpResult` on failed live write validation; endpoint-level middleware still requires `SystemAdministrator`.
- Regression workflow exercises an actual RoleCreate path with migrated SQL Server and rejects invalid master RolePermission target with 403.
- Security caveat: coarse SystemAdministrator continues to authorize these operations until a controlled Resource.Action catalog with grants and upgrade path exists. Logging actor and operation does not by itself constitute a durable tamper-evident audit ledger.

## Verified latest head — 2026-10-23

- Backend [run 39868009455](https://github.com/sholehamed/Shinera-backend/actions/runs/39868009455) succeeded: Release Build and **21/21 xUnit**.
- Disposable SQL Server [run 39868009445](https://github.com/sholehamed/Shinera-backend/actions/runs/39868009445) succeeded: real migrations, **12/12 integration tests**, and application startup/seed/admin login with protected Roles GET plus Role POST (201) and out-of-scope permission PUT (403).
- Security hardening also tightened `ActiveSystemAdministratorHandler` to require active master-scoped SuperAdmin assignment, not a lookalike role granted by an arbitrary tenant. Existing SQLite/SQL Server fixtures were updated to reflect this invariant; interim failing checks are not presented as final results.
- All changes are in the new feature branch. **No merge has been performed.** Complete Permission catalog CRUD, PUT/DELETE/PATCH HTTP integration and durable authorization audit remain Issue #11 follow-ups.


## Latest additional hardening and CI — 2026-10-23

- Added `AuthorizedPlatformMasterWrite` as a separately testable, live-database-validated master write gateway. It verifies the current System actor, active account, active Master Tenant and a master-scoped SuperAdmin membership *before* opening the bounded write scope. The gateway restores the read filter and write scope together after disposal, and logs authorization failures and scope lifecycle. **The current Role HTTP endpoints still use `PlatformRoleWriteExecutor`; consolidation of these two internal gateways should be considered before production readiness.**
- Added a SQL Server regression that confirms active authorized master-scope write succeeds, other-tenant writes fail, the scope does not leak after disposal, revoking the persisted administrator role blocks access despite stale JWT claims, and a forged `tenant_id` claim is rejected.
- Latest [Backend CI 39869241494](https://github.com/sholehamed/Shinera-backend/actions/runs/39869241494): **21/21 xUnit passed** with Release Build.
- Latest [SQL Server CI 39869241918](https://github.com/sholehamed/Shinera-backend/actions/runs/39869241918): **13/13 integration tests passed**, plus live server migration/seed/login, authorized Role POST and forbidden cross-scope write.
- Changes remain on branches and PRs; **no merge**. Issue #11 has remaining per-action fine-grained permission grants, complete Permission CRUD and transaction/rollback tests.
