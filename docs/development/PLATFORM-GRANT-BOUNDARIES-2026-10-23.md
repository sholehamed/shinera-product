# Shinera — Target-scoped platform grant write boundaries

**Date:** 2026-10-23  
**Status:** In progress / draft, not merged  
**Backend:** [PR #16](https://github.com/sholehamed/Shinera-backend/pull/16) — `security/platform-grant-boundaries-20261023`  
**Tracking:** [P0 Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11) and [ADR-003](../architecture/decisions/ADR-003-AUTHORIZATION-AND-PERMISSION-MODEL.md)

## Review finding and threat model

A validated System SuperAdmin previously received a temporary Master write scope through `AuthorizedPlatformMasterWrite`. That bounded scope restricted *TenantId*, not *what type of authorization grant could be written*. If any allowed application command or shared tracked DbContext staged an unrelated `UserRole`, `GroupRole`, `UserPermission`, `UserGroup` or `RolePermission` mutation before `SaveChanges`, its legitimate scope could inadvertently persist an out-of-band privilege change.

A writable Master TenantId is **not itself** permission to change every grant or assign arbitrary roles. Read filter bypass and trusted master write scope are not universal permission bypasses.

## Implemented reviewable security slice

- `TrustedTenantWriteScope` now holds a one-operation, disposable grant intent within an already validated platform master write. It only supports `RolePermissions` for one `RoleId` and `GroupMembers` for one `GroupId`.
- `TenantSaveChangesInterceptor` rejects master-scoped `UserRole`, `GroupRole` and `UserPermission` mutations during ordinary platform operations. They require separate explicitly designed authorization APIs and migration policy.
- `RolePermission` mutations are allowed **only** when the existing `PlatformRoleWriteExecutor` runs the `roles.permissions.replace` operation against that exact verified target role.
- `UserGroup` mutations are allowed **only** in `PlatformGroupMembershipWriteExecutor` for the validated active Master group. Unrelated group writes are denied even during a valid group operation.
- Grant intents are disposed on normal and exceptional exits. The existing Development/bootstrap bypass and ordinary Tenant-scoped write behavior are unchanged.

## Verification gate

New SQL Server regression verifies that a general trusted master scope cannot create direct platform role/group/user permission grants or unscoped role/group links, and that authorized role/group operations reject cross-target grant smuggling. Existing positive RolePermission and UserGroup tests provide regression coverage for supported operations.

- [Build/xUnit](https://github.com/sholehamed/Shinera-backend/actions/runs/39914356700): pending final status verification.
- [Real SQL Server authorization](https://github.com/sholehamed/Shinera-backend/actions/runs/39914356804): pending final status verification.
- Do not claim post-merge verification or Production rollout from PR-head CI alone.

## Explicit remaining P0 work

1. Add properly authorized, target-aware, independently audited workflows for the intentional creation/revocation of `UserRole`, `GroupRole`, `UserPermission` (currently blocked in master request scope).
2. Define self-escalation, separation-of-duties, privileged role bootstrap, and delegated administrator policy; verify group-inherited and direct permission semantics.
3. Close transactional revocation and check-then-use concurrency races, add durable tamper-evident audit, test full positive/negative HTTP matrix, and review raw/bulk SQL bypass paths.
4. Prepare and review a one-time Production permission/role grant migration with rollback. Keep `EnforceRoleEndpoints` and `EnforcePermissionEndpoints` **OFF** in Production until reviewed grants exist. Never enable Development bootstrap grants in Production.

**Scope:** This is an incremental security boundary, **not completion of Issue #11**. No Production authorization switches are changed.
