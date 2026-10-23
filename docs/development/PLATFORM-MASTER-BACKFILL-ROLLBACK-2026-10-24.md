# P0 Stage 5B — Explicit Master grant backfill and exact rollback

**Status:** Reviewed and merged offline-only implementation slice. NOT a Production deployment approval  
**Backend:** [PR #30](https://github.com/sholehamed/Shinera-backend/pull/30) · [merged commit `1c4700a0`](https://github.com/sholehamed/Shinera-backend/commit/1c4700a0910ae80630ae3e4fee651c7ae57ce318)  
**Tracking:** [Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11)  
**Prior gate:** [Stage 5A reviewed inventory](PLATFORM-MASTER-PRODUCTION-CUTOVER-2026-10-24.md)

## Scope of this slice

The backend now contains a stand-alone **offline-only** `PlatformMasterGrantBackfill` primitive. It is NOT registered as a service or endpoint, is not called by seeding or startup, and has **no Production execution path** in this slice. An independently authenticated operator, approval-signature verification, deployment orchestrator and recovery process have **not** been implemented. There is no implicit permission grant to all existing System administrators.

The primitive deliberately supports only **explicit RolePermission additions** to one already-existing, live, fixed-Master `SUPERADMIN` role. It does **not** create roles, System accounts, UserRoles, GroupRoles or direct UserPermissions; it never sets `User.TenantId`. This is a constrained first part of Production grant backfill, not complete general-purpose permission migration.

## Reviewed manifest

The input `Manifest` requires:

- Exactly the fixed Master Tenant ID, the target Master SuperAdmin Role ID, and the actual target database name
- Distinct actor and reviewer System user IDs, both currently active, unlocked and **already assigned** the live Master SuperAdmin role
- The **complete exact before-image** of that Role's grants, including every pre-existing `RolePermission.Id` and `PermissionId`
- At most nine precisely identified, nonempty additions, with **preapproved new RolePermission IDs** and existing active platform-catalog Permission IDs; no duplicated target/assignment IDs

The executor uses a SQL Server `Serializable` transaction, role-specific `sp_getapplock`, live Master/User/Role/Permission validation, strict before-image comparison and parameterized SQL. **Any mismatch fails closed**, rather than computing a new privilege proposal from the current database. On each grant it writes the RolePermission and an entry in existing append-only `PlatformGrantAuditEntries` **inside the same transaction**.

Rollback accepts the exact same manifest, requires the entire expected after-image to match, and removes **only those exact new Row IDs** if they still match role, permission and `CreatedBy`. Before deletion it also requires a previously committed append-only `grant` audit for the fixed Master scope, actor, role and permission; otherwise it fails closed, **even if the row ID and creator happen to match**. It appends compensating `revoke` audit rows in the same transaction. A repeat apply/rollback fails closed; there is no force-overwrite option. These checks reduce accidental revocation risk, but because the historical ledger does **not** capture `RolePermission.Id` or `BatchId`, they cannot prove row-level batch provenance.

## Limitations and controls not to skip

The existing grant audit schema has no immutable `BatchId`, signer identity, approval hash or `RolePermissionId` link. A row's matching `Id`, `PermissionId` and `CreatedBy` therefore provides **operational targeting**, not a tamper-proof proof of batch provenance. Immutable separately held manifest/receipts, signature verification, database access controls and reconciliation must be implemented and reviewed before deploying this primitive.

A separate production-only offline operator tool, fully reviewed two-person approval process, secure batch artifact storage, resuming/reconciling ambiguous DB timeouts, permission catalog migration readiness, effective-grant evaluation and manual disaster recovery exercises remain **P0 acceptance gates**. SQL application locks coordinate only cooperating writers; the serializable snapshot protects this transaction but is not a replacement for maintenance-window/change freeze controls. Database Admin and DDL principals can alter audit triggers.

**No Production release authorization is implied by merging the code.** Granular enforcement and the `AllowCodeUniquenessMigration` flag remain OFF unless separately approved.

## SQL Server regression plan

- Exact approved single-grant addition adds one RolePermission row and one append-only audit event.
- Reapplying the identical manifest fails because before-image has changed; the existing unrelated grant remains untouched.
- Exact rollback checks committed grant-audit evidence, removes only the new Row ID and appends one `revoke` audit entry. Second rollback is denied.
- A fabricated rollback pointing at a manually inserted row with matching actor/ID/permission, but no prior grant-audit record, is rejected without deleting the row.
- Unexpected unrelated RolePermission between approval and execution blocks apply with zero new writes.
- Simulated SQL audit insert failure causes the grant insert to roll back atomically; an explicit later replay is possible.
- With two grants in one batch, a failure on the **second audit insert** must undo both RolePermission rows and the first audit insert (verified in a separate SQL regression).
- A wrong target database name and revoked independent reviewer both fail without a new RolePermission or audit row.
- Existing SQL and OpenIddict real HTTP suite must remain green; no endpoint or authorization flag changes.

**Next:** finalize fail-closed review and CI, then add an operator-signed batch receipt/ledger, independently verified approval, exact row attribution, recovery orchestration and Production rollout rehearsals before Stage 5C cutover/ADR-003 acceptance.

## CI evidence (backend PR-head only)

Final Backend PR head `e7b24cc3e7115470d5f230b2850ba26bdabc090e` passed [Build/Unit run 40030147984](https://github.com/sholehamed/Shinera-backend/actions/runs/40030147984) and [real SQL Server and signed OpenIddict HTTP run 40030148013](https://github.com/sholehamed/Shinera-backend/actions/runs/40030148013): **33/33 SQL Server tests and 89/89 real HTTP checks**. Backend PR #30 was reviewed and merged at [`1c4700a0`](https://github.com/sholehamed/Shinera-backend/commit/1c4700a0910ae80630ae3e4fee651c7ae57ce318). This is final PR-head disposable SQL Server CI evidence, **not** a Production rehearsal, cryptographically approved operator execution mechanism, or separate post-merge SQL Server CI. The actual Production backfill/cutover is **not** approved.
