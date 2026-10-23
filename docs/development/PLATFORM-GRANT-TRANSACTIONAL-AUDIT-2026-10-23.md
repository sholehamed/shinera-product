# Platform grant transactional audit — first durable ledger slice

**Date:** 2026-10-23  
**Status:** Draft; [Backend PR #20](https://github.com/sholehamed/Shinera-backend/pull/20)  
**Tracking:** [Backend P0 Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11)  
**Backend branch:** `security/platform-grant-transactional-audit-20261023`

## Scope

Three previously reviewed System-only, Master-scoped write services now insert an immutable SQL record **inside the same SQL Server transaction and before the grant transaction commits**:

- `PlatformUserRoleWriteService`: role granted or revoked
- `PlatformGroupRoleWriteService`: group role granted or revoked
- `PlatformUserPermissionWriteService`: explicit Allow, Deny, or Clear (remove override)

Exactly one record per successful changed operation. Idempotent no-ops and access denials do not generate an audit event, even when the request was attempted; independent **denied-attempt security logging** and incident monitoring remain separate work.

## SQL schema

Manual EF migration `20261023110000_platformgrantaudit` creates `dbo.PlatformGrantAuditEntries`. Each entry contains a distinct Guid `Id`, fixed `TenantId` Master scope, `ActorUserId`, `Kind` (`UserRole`, `GroupRole`, `UserPermission`), `TargetId`, `RelatedId`, `Action`, nullable `PreviousGrant`/`CurrentGrant`, and server UTC `OccurredAtUtc`. The target/time index supports future authorized history reads.

No foreign keys reference users, groups or roles: audit rows must survive business-row deletion. The table is intentionally **not an EF tracked DbSet**, and the migration creates an SQL trigger to reject direct `UPDATE` or `DELETE` of audit rows. The raw migration is managed explicitly; any future migration baseline/snapshot generation must account for this SQL-owned table.

## Atomicity

The recorder refuses to run without `CurrentTransaction`, uses EF-parameterized SQL rather than string concatenation, and inserts before the grant transaction commits. For an Allow-to-Deny flip, both grant-row SaveChanges and the audit insert participate in the *same* serializable transaction; failure of any statement causes rollback. Audit events capture old/new state; no-op requests generate no record. The grant authorization policy, exact scoped grant intent and Production enforcement flags are unchanged.

## Security limitations (critical)

**Append-only is not tamper-evident.** A DB principal with DDL rights can disable/drop triggers and alter or destroy the records. To reach integrity assurance, plan separate restricted DB credentials, DDL event monitoring, chained hashes or signatures with externally protected keys, secure off-server immutable export, and backup/retention controls. Migration Down drops the table; production rollback needs independent preservation and explicit approval.

This slice does not cover historical backfill, `UserGroup` membership, `RolePermission` changes, bulk/raw SQL paths, unauthorized access attempts, or privileged bootstrap operations. A review-only Audit API, read authorization, and data minimization/retention decisions remain open.

## Verification

Real SQL Server regressions assert that successful UserRole and GroupRole grant/revoke have one audit record per change; UserPermission Deny, Allow, Clear and explicit Allow generate audit entries; no-ops and denied requests do not. They also assert audit SQL `UPDATE` and `DELETE` are rejected.

Build/Unit and SQL Server workflows on the **final backend PR head** must be checked before merge. Passing CI is not post-merge or production validation. Do not close Issue #11 or turn on granular Production flags.
