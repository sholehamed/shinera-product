# Shinera — System UserPermission override administration

**Date:** 2026-10-23  
**Status:** In progress / Backend [Draft PR #19](https://github.com/sholehamed/Shinera-backend/pull/19)  
**Branch:** `security/platform-user-permission-grants-20261023`  
**Tracking:** [Identity P0 Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11), [ADR-003](../architecture/decisions/ADR-003-AUTHORIZATION-AND-PERMISSION-MODEL.md)

## Scope and semantics

This slice exposes direct UserPermission overrides for null-tenant **System** accounts, stored in the fixed Master Tenant. It is not a tenant-user/branch/own/child grants API.

With `BackendPrefix=/api`:

```http
PUT     /api/System/Users/{id}/permissions/{permissionId}/allow
PUT     /api/System/Users/{id}/permissions/{permissionId}/deny
DELETE  /api/System/Users/{id}/permissions/{permissionId}
```

The first path inserts or replaces an **Allow** (`IsGranted=true`); the second inserts or replaces a **Deny** (`IsGranted=false`); DELETE **removes** the direct override, falling back to currently effective role/group inheritance. DELETE does not create a Deny. Routes are idempotent, and successful operations return 204. The caller never controls `TenantId` in a request body.

## Effective permission precedence

The existing `SystemPermissionHandler` evaluates only authenticated System accounts in the Master grant scope:

1. Inactive/locked/foreign principal and inactive Master state are denied.
2. An active explicit `UserPermission.IsGranted=false` for the requested stable `Resource.Action` permission key **DENIES**, even if a UserRole and/or GroupRole would allow it.
3. Otherwise a direct active `UserPermission.IsGranted=true` **ALLOWS**.
4. Otherwise an active UserRole permission or active GroupRole reached through an active Group **ALLOWS**.
5. All other requests default to denied.

This permission check is **additional to** the coarse `SystemAdministrator` role requirement for sensitive System endpoints, where enforced. The new routes inherit that coarse requirement; these changes **do not enable** the staged fine-grained policies in Production.

## Mutation constraints

- Live, persisted active System SuperAdmin authorization is rechecked in the write gateway **within** the same SQL Server Serializable transaction as mutation. JWT role claims alone are insufficient.
- Only another System user with `User.TenantId=null` may be targeted. No self overrides. **Target accounts already assigned a Master SuperAdmin role are protected**, including removal of overrides.
- New Allow and Deny require an active, non-locked target and active permission. Removal remains possible for an inactive target/permission to support cleanup.
- No HTTP overrides for reserved `ROLES.*` or `PERMISSIONS.*` codes, in either direction. These catalog privileges require reviewed deployment and policy.
- The interceptor permits writing `UserPermission` only while a disposable intent binds the precise `UserId + PermissionId` pair inside a verified Master write. Other UserRole/GroupRole/UserGroup/RolePermission rows remain protected.
- Changing Allow ↔ Deny removes and recreates the same unique grant row via two guarded SaveChanges **inside one database transaction**. Partial changes roll back if the second write fails. In-place grant edits remain forbidden.
- Structured actor, subject, permission and action logging is emitted for changed grants, but this **does not constitute durable/tamper-evident audit**.

## Regression evidence (verified on PR head)

New real SQL Server test covers inherited authorization through both Role and Group, Deny precedence, Allow restoration, removal restoring inherited behavior, idempotency, no inherited Export/explicit Allow, self, tenant, SuperAdmin, reserved catalog refusal, piggybacked grant rollback, and revoked administrator grant refusal.

[Backend Build/Unit run 39921680594](https://github.com/sholehamed/Shinera-backend/actions/runs/39921680594) **passed** and [SQL Server authorization run 39921680707](https://github.com/sholehamed/Shinera-backend/actions/runs/39921680707) **passed** on exact backend PR head `e71a1a19b9538cd5dcbc0a415c6df3e1eeeecb36`. These are feature-branch checks, not Production or post-merge verification. Final security review and Merge remain separate.

## Deferred P0 requirements

Durable immutable grant audit; HTTP 401/403/204 real-auth matrix; concurrency/lock/retry contention tests; privilege-review workflows; complete Group/user migration and rollback; permission catalog/separation-of-duties review, and a separate Production cutover. All Production granular authorization flags stay OFF.

Do **not** close [Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11) after this partial slice.
