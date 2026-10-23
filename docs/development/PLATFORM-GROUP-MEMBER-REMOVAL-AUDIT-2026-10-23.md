# Platform System group membership revocation — transactional audit

**Date:** 2026-10-23  
**Status:** Merged [Backend PR #23](https://github.com/sholehamed/Shinera-backend/pull/23) into backend main at [`0b791934`](https://github.com/sholehamed/Shinera-backend/commit/0b7919344c2ad8bb62683daa774b26ccf5b5b336); Issue #11 remains OPEN  
**Backend branch:** `security/platform-group-member-removal-audit-20261023`  
**Tracking:** [Backend P0 Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11)

## API and authorization

Under `BackendPrefix=/api` the newly introduced route is:

```http
DELETE /api/System/Groups/{groupId:guid}/members/{userId:guid}
```

An authenticated `SystemAdministrator` is required at the endpoint group and independently rechecked against an **active, persisted System SuperAdmin membership** by the Master write gateway within the mutation transaction. A caller cannot specify a storage TenantId. Both route parameters are required, and success (including an already-absent association) returns HTTP 204.

The user may only be a null-tenant System subject, never a tenant account. The group must belong to the fixed Master Tenant, and a group linked to a reserved `SUPERADMIN` role is never editable by this ordinary workflow. An actor cannot revoke their own membership. Revocation is permitted on inactive groups, locked/inactive System members to clean up stale privileges. Invalid, privileged and foreign subjects fail closed (403).

## Exactly scoped, atomic grant revocation

The new `PlatformGrantMutationKind.GroupMemberRemoval` authorizes one `UserGroup` entity only when **both** `GroupId` and `UserId` equal the reviewed URI target. The existing SaveChanges interceptor rejects any same-group but unrelated member deletion or other grant mutation piggybacking onto this operation.

The service wraps the **whole sequence** in EF Core's SQL Server execution strategy and `Serializable` transaction: live SuperAdmin verification; group, reserved role, and target checks; deletion of the existing `UserGroup` row; parameterized `PlatformGrantAuditEntries` insertion with `Kind=UserGroup`, `Action=revoke`, `PreviousGrant=true`, `CurrentGrant=false`; and transaction commit. If SQL audit insertion fails, no membership is removed. Absent links are semantic no-ops and do not generate change events.

Audit records remain append-only for ordinary SQL DML. This does **not** defend against a DBA with DDL privileges and is not cryptographically tamper-evident.

## Verification

Real SQL Server integration tests extend the group membership test to cover protected group/foreign user/self restrictions, successful and idempotent revoke, failed audit insertion rolling back a deletion, unrelated tracked membership rollback, cleaning up an inactive System member, and a revoked administrator token. [Build/Unit CI 39932889770](https://github.com/sholehamed/Shinera-backend/actions/runs/39932889770) and [real SQL Server authorization CI 39932889728](https://github.com/sholehamed/Shinera-backend/actions/runs/39932889728) both **passed** on exact backend PR head `947da057838642c72f28b00c9c1c9ac5a9583083`, including SQL regression and API startup smoke. The backend was squash-merged at [`0b791934`](https://github.com/sholehamed/Shinera-backend/commit/0b7919344c2ad8bb62683daa774b26ccf5b5b336) after final security review. CI above is final PR-head evidence, **not independent post-merge SQL Server or Production verification**.

## Remaining Issue #11

This step does not implement batch removal, an authenticated HTTP 401/403/204 E2E matrix, concurrency stress, audit viewer authorization, signatures/external sealing, privileged bootstrap audit, or a Production-safe permission migration/cutover. Production granular permission flags remain **OFF**; [Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11) stays open.
