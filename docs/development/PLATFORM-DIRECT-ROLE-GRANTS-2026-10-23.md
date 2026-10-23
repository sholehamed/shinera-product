# Shinera — Direct System Role grant/revoke (first secure slice)

**Date:** 2026-10-23  
**Status:** Merged backend security slice [PR #17](https://github.com/sholehamed/Shinera-backend/pull/17) at [`52571097`](https://github.com/sholehamed/Shinera-backend/commit/52571097ea5bee2c19913982fb0d0e5068e9f7da); Production enforcement remains OFF  
**Tracking:** [P0 Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11), [ADR-003](../architecture/decisions/ADR-003-AUTHORIZATION-AND-PERMISSION-MODEL.md)  
**Backend branch:** `security/platform-direct-grant-administration-20261023`

## Scope

A dedicated, narrowly-scoped path to **grant and revoke non-SuperAdmin master-scoped Roles** to other System accounts. This is NOT a general grant API or tenant-user assignment feature.

The `Users` System endpoint group inherits `SystemAdministrator`. With configured `BackendPrefix`, its new role routes are:

```http
PUT    /api/System/Users/{id}/roles/{roleId}
DELETE /api/System/Users/{id}/roles/{roleId}
```

Requests have no mutable TenantId payload; role and target identifiers are route-bound. Both are idempotent and respond with 204 for authorized requests, including already granted or absent assignment.

## Security decisions

1. The gateway rechecks an authenticated System actor's **live**, active Master `SUPERADMIN` membership, active account, lockout and Master state inside the database transaction, rather than trusting stale JWT claims.
2. The target must be a separate System account with **null `User.TenantId`**. For new grants, the target must be active and not locked out. Revocations remain available for inactive accounts.
3. The role must exist in the fixed Master role storage scope. Granting disabled roles is forbidden. **Any grant or revocation of the `SUPERADMIN` role is forbidden in the HTTP workflow**, including attempts to rotate or remove another person's administrator assignment. This remains a privileged migration/bootstrap operation.
4. An exact, temporary `UserId + RoleId` intent is required inside the verified master scope. The existing `TenantSaveChangesInterceptor` refuses `UserRole` mutation outside that exact pair, and refuses in-place grant reassignment (revoke then assign instead).
5. Validation and write occur inside one SQL Server `Serializable` transaction, with EF Core's retry execution strategy managing the entire transaction, not just SaveChanges.
6. Security-relevant `grant` / `revoke` outcomes produce structured logger entries with actor, target user and role IDs. **Logging alone is not durable, tamper-evident audit history.** A dedicated audit design remains open.

## Verification

SQL Server integration scenarios were added for valid, idempotent grant/revoke; self-escalation refusal; reserved SuperAdmin role refusal; tenant or foreign-scope denial; indefinite target lockout refusal; attempts to piggyback unauthorized GroupRole grants into the same SaveChanges; denied operations preserving existing grants; and post-revocation stale admin credential refusal.

Final backend PR head `6c1d600a93285f2bce7035750c6a63c0a2873928` passed [Build/Unit CI 39916928238](https://github.com/sholehamed/Shinera-backend/actions/runs/39916928238) and [SQL Server authorization CI 39916927938](https://github.com/sholehamed/Shinera-backend/actions/runs/39916927938), then was squash-merged at [`52571097`](https://github.com/sholehamed/Shinera-backend/commit/52571097ea5bee2c19913982fb0d0e5068e9f7da). These are **PR-head checks**, not a separate post-merge SQL Server or Production verification.

## Deferred / P0 remaining

- Explicit reviewed `GroupRole` and `UserPermission` grant and revoke workflows, including precedence of direct deny over role/group allow.
- Separation-of-duties and delegated administrator authorization, potential `roles.assign-users` catalog policy and staged Production permission migrations. Current live `SystemAdministrator` is necessary for these routes and no granular policy is newly activated.
- HTTP-level 401/403/204 E2E with real authentication; concurrent add/remove stress and token revocation propagation.
- Transactional durable audit, controlled production bootstrap, migration and rollback. No automatic Production privileged grant is introduced.

This is an incremental P0 issue slice: **do not close Issue #11 or enable Production granular permissions** after merging it.
