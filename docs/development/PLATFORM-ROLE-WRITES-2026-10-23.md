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
- The endpoint path must return a **403** on failed live authorization, not a generic 500; validate this with real HTTP integration tests.
- Tests must verify the actual ASP.NET Core POST/PUT/DELETE/PATCH endpoints against the disposable SQL Server (as well as service/EF tests). Positive role creation and forbidden foreign-role mutation must be proven before merge.
- The System user cannot select an arbitrary TenantId, and the master scope remains a legacy storage detail, not a parent tenancy.
- Role writes performed outside this executor must still fail through the interceptor. Bypass via EF bulk or raw SQL is a separate audit item.
- [Issue #9](https://github.com/sholehamed/Shinera-backend/issues/9) remains open until all tenant-write acceptance criteria and bypass reviews are completed.

## CI and verification ledger

| Test | Status | Evidence |
|---|---|---|
| Backend Build + unit tests | Pending latest head validation | [Backend PR #12 checks](https://github.com/sholehamed/Shinera-backend/pull/12/checks) |
| SQL Server migration/role/assignment tests | Pending latest head validation | [Backend PR #12 checks](https://github.com/sholehamed/Shinera-backend/pull/12/checks) |
| API-level role CRUD and authorization HTTP statuses | Not yet fully exercised | Add E2E cases |
| Production migration/role grant bootstrap | Not implemented | Requires separate rollout decision |

## Recorded implementation issues

- Initial Build failed because `System.Text.Json.Serialization.JsonIgnore` in System-prefixed namespaces resolved to a project namespace. Fixed by using `global::System.Text.Json.Serialization.JsonIgnore`, with new CI run.
- Keep all PRs Draft until latest CI and complete security regression review.
