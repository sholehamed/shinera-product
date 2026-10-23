# Shinera — Restricted platform GroupRole grant and revoke

**Date:** 2026-10-23  
**Status:** Draft security slice / CI review pending  
**Backend:** [Draft PR #18](https://github.com/sholehamed/Shinera-backend/pull/18) (`security/platform-group-role-grants-20261023`)  
**Tracking:** [Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11), [ADR-003](../architecture/decisions/ADR-003-AUTHORIZATION-AND-PERMISSION-MODEL.md)

## Purpose and HTTP surface

Extend the previous restricted direct `UserRole` workflow with intentionally narrow `GroupRole` assignment/revocation for *platform groups and non-SuperAdmin roles only*.

Under the configured `BackendPrefix` (the examples use `/api`):

```http
PUT    /api/System/Groups/{groupId}/roles/{roleId}
DELETE /api/System/Groups/{groupId}/roles/{roleId}
```

Both are protected by the existing live `SystemAdministrator` requirement on the Groups endpoint group and **again** within the database-backed write gateway; success and idempotent no-op return HTTP 204. Group and role identifiers are URI values. Clients do not supply or change the storage TenantId.

## Security boundary

- Actor must remain an active null-tenant System user with a persisted active `SUPERADMIN` role link in the fixed Master scope; stale JWT roles do not grant platform write authority.
- The group and role must belong to the **Master Tenant**, and assignments additionally require both to be active. Existing non-privileged assignments can still be revoked from disabled groups/roles.
- Assignment and revocation of `SUPERADMIN` via this API are both denied. **Any group already linked to a SuperAdmin role** is also protected from generic GroupRole mutation.
- Self-affecting operations are denied if the acting administrator belongs to the target group, avoiding indirect self-granted privileges.
- **Reverse-order self escalation is denied too:** the existing GroupMembers endpoint now refuses an administrator adding themselves to any platform group, including one that already holds a non-SuperAdmin role. The SQL Server regression tests this sequence explicitly.
- Before granting a role, all current members must be active, unlocked, null-tenant **System** users. An empty group is valid. Inactive, locked-out, tenant-owned or otherwise ineligible group members make new assignment fail closed.
- `TrustedTenantWriteScope` now has a dedicated temporary `GroupRoles` intent for an **exact `GroupId + RoleId` pair**. `TenantSaveChangesInterceptor` rejects any unrelated or in-place modified master authorization grant even within this temporary Master write scope. Direct `UserPermission` writes stay blocked.
- Actor revalidation, read of group/role/membership and grant insertion/deletion occur under one SQL Server `Serializable` transaction, with the EF Core execution strategy retrying the complete unit.
- Structured logging includes operation, actor, group and role. **Application logs are not a tamper-evident durable audit.**

## Test scope and review gates

SQL Server regression test cases include authorized grant, duplicate grant, revoke and duplicate revoke; SuperAdmin role protection and privileged group protection; actor member/self-escalation; cross-tenant group/role, invalid member and inactive group denial; rollback on piggybacked UserRole mutations; and persisted administrator role revocation.

The previous [Build/Unit run 39919612264](https://github.com/sholehamed/Shinera-backend/actions/runs/39919612264) and [SQL Server run 39919612059](https://github.com/sholehamed/Shinera-backend/actions/runs/39919612059) passed on the original PR head. The security review discovered a **two-step administrator self-join escalation**, which has been patched on this branch with a new SQL Server regression. **Final updated PR-head CI must pass before merging; earlier green checks do not verify the fix.**

## Deferred security tasks

- Direct `UserPermission` grant/revoke, explicit deny semantics and privilege inheritance analysis.
- Durable, transactional tamper-evident grant audit and detailed HTTP 401/403/204 E2E coverage.
- Concurrency and lock-contention tests beyond sequential integration, review of legacy Group CRUD and indirect grants.
- Production-safe role/permission bootstrap, reviewed migrations and rollback. **Production granular authorization flags remain OFF.**

This is an incremental [Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11) slice. Do not mark the issue complete after merging.
