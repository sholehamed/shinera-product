# Platform grant audit — Group membership and Role permission deltas

**Date:** 2026-10-23  
**Status:** Draft implementation, [Backend PR #21](https://github.com/sholehamed/Shinera-backend/pull/21)  
**Tracking:** [P0 Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11)  
**Backend branch:** `security/platform-membership-rolepermission-audit-20261023`

## Summary

This incremental phase extends the existing SQL Server append-only `PlatformGrantAuditEntries` ledger, delivered in [merged backend PR #20](https://github.com/sholehamed/Shinera-backend/pull/20), to two previously unaudited verified System-only Master grant paths:

- `UserGroup` changes made by `POST /api/System/Groups/{groupId}/addmembers`, for each actual inserted member. The executor determines the actual persisted GroupId/UserId membership set before and after execution, recording only differences.
- `RolePermission` changes made by `PUT /api/System/Roles/{roleId}/permissions`, recording a separate grant or revoke for each actual permission ID added or removed. This is a replacement-of-set request, but audit represents its *effective delta*, not a noisy event for every requested permission.

Audit entries use the existing verified actor ID, Master `TenantId`, `Kind`, `TargetId`, `RelatedId`, `Action`, previous/current grant booleans, and database UTC timestamp. `UserGroup` uses GroupId as TargetId, UserId as RelatedId; `RolePermission` uses RoleId as TargetId, PermissionId as RelatedId. No database schema change or new public audit-read API.

## Atomicity, authorization, and idempotence

The verified gateway, before/after snapshot, existing CQRS handler, and one audit INSERT per changed link now execute within **one SQL Server Serializable transaction**. EF execution-strategy retry encloses the entire transaction. An audit INSERT failure rolls back grant writes. This extends the existing master-write and grant-intent protections, *not* a separate authorization or privileged grant path.

The RolePermission handler now computes set differences before SaveChanges; identical sets no longer delete/reinsert grant links. Repeated AddGroupMembers requests likewise yield no new member link and no audit event.

A `UserGroup` revoke HTTP endpoint is not added in this phase. If an existing guarded group membership operation changes a link, the executor's persisted delta still records its removal. Untrusted direct/bulk SQL and generic operations outside these verified executors are **not** claimed to be covered.

## Verification

SQL Server tests added or strengthened to check correct event counts, actor attribution, duplicate/no-op suppression, permission revocation and re-grant, denied operations, and rollback after a temporary SQL trigger deliberately fails audit INSERTs. Final backend PR head `e7bf0696be111c27cc008159011a80ff73133589` passed [Build/Unit CI run 39926918483](https://github.com/sholehamed/Shinera-backend/actions/runs/39926918483) and [real SQL Server authorization CI run 39926918475](https://github.com/sholehamed/Shinera-backend/actions/runs/39926918475), including audit INSERT fault-injection rollback tests. These are exact PR-head results, not independent post-merge or Production verification.

## Known limits

The existing database append-only trigger prevents SQL UPDATE/DELETE of audit rows, but a DBA with DDL rights can disable it. **This is not tamper-evident auditing.** Privileged SQL DDL monitoring, off-server integrity sealing, audit-read authorization, retention, privileged bootstrap actions, remaining legacy writes, concurrency and full authenticated HTTP 401/403/204 tests remain part of Issue #11.

**Production granular authorization remains OFF.** No Production schema migration or privilege rollout is introduced by this slice. Keep [Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11) open.
