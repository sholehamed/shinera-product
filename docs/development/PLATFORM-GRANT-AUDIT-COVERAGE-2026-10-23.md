# Platform grant audit coverage — GroupMembers and RolePermissions

**Date:** 2026-10-23  
**Status:** Reviewed and merged backend [PR #22](https://github.com/sholehamed/Shinera-backend/pull/22) into main at [`895fa89f`](https://github.com/sholehamed/Shinera-backend/commit/895fa89fa1204bc55d86a17494e399f8693b4650)  
**Issue:** [Identity P0 #11](https://github.com/sholehamed/Shinera-backend/issues/11)  
**Branch:** `security/platform-grant-audit-coverage-20261023`

## Scope

Extends the merged SQL append-only `PlatformGrantAuditEntries` ledger to the two existing System Master-scoped administrative workflows that were not covered by the first audit slice.

**Group membership (UserGroup):** `Groups/{GroupId}/addmembers` checks a live database-backed System SuperAdmin, an active Master group, nonprivileged group membership, and eligible null-tenant System accounts. Validated additions and per-added-member audit records now share one SQL Server Serializable transaction. Repeated member IDs and already existing memberships produce no grant-change audit. Self membership and SUPERADMIN groups remain protected.

**Role permissions (RolePermission):** `Roles/{roleId}/permissions` requires a verified System administrator and Master role, prohibits edits to the SUPERADMIN role, and compares persisted permission assignments before/after the legacy replacement command. Each effective addition or removal produces a grant/revoke audit event within the **same Serializable transaction**. Replacements with an unchanged permission set do not produce change events, even though the legacy command may rewrite rows.

Both paths execute inside EF Core's SQL Server retry execution strategy, with live authorization checks **inside** the transaction rather than before it.

### Final-review security hardening

The membership executor snapshots **all** current members of the target group before and after the legacy handler. If the delta includes a non-requested membership, an unrequested deletion, or misses any expected validated addition, it fails closed and rolls the transaction back. This blocks unrelated pre-tracked changes to the *same* group that previously matched the broad group grant intent.

The role-permission endpoint passes its exact requested permission IDs to the transactional writer. After the command saves its assignment changes, the writer verifies that the persisted set matches that reviewed list exactly. A same-role, pre-tracked permission cannot silently piggyback into the committed state. Failures roll back the replacement and all audit entries.

Real SQL Server regressions explicitly stage these same-target piggyback mutations and verify no unauthorized data or audit entry persists. The stored actor and exact group/role, member/permission, event type, and old/new assignment states support later authorized audit views.

## Verification and deployment boundary

Real SQL Server integration regression tests exercise idempotent membership changes, changed role-permission sets, injected audit INSERT failures rolling back the corresponding grant, and recovery. [Build/Unit CI 39931302854](https://github.com/sholehamed/Shinera-backend/actions/runs/39931302854) and [SQL Server authorization CI 39931303035](https://github.com/sholehamed/Shinera-backend/actions/runs/39931303035) **passed** on exact final reviewed PR head `28bc0e2ed0abbf4e956580999f741ac1b30d8f2d`, covering the rollback and same-target piggyback regressions. Backend was squash-merged at [`895fa89f`](https://github.com/sholehamed/Shinera-backend/commit/895fa89fa1204bc55d86a17494e399f8693b4650). These are feature-branch checks, not independent post-merge or Production validation.

This is a partial P0 Issue #11 slice. It does **not** add member removals, bypass-proof bulk/legacy grant mutation capture, privileged bootstrap auditing, cryptographic tamper evidence against DBAs, a read-only Audit API, the complete authenticated 401/403/204 HTTP matrix, or approved Production deployment. Production granular enforcement flags remain OFF. The ledger's SQL UPDATE/DELETE trigger offers append-only behavior for ordinary database DML but is not tamper-evident against DDL-capable principals.

No production rollout or closing Issue #11 is authorized by this PR.
