# Platform grant audit coverage — GroupMembers and RolePermissions

**Date:** 2026-10-23  
**Status:** In progress — [backend Draft PR #22](https://github.com/sholehamed/Shinera-backend/pull/22)  
**Issue:** [Identity P0 #11](https://github.com/sholehamed/Shinera-backend/issues/11)  
**Branch:** `security/platform-grant-audit-coverage-20261023`

## Scope

Extends the merged SQL append-only `PlatformGrantAuditEntries` ledger to the two existing System Master-scoped administrative workflows that were not covered by the first audit slice.

**Group membership (UserGroup):** `Groups/{GroupId}/addmembers` checks a live database-backed System SuperAdmin, an active Master group, nonprivileged group membership, and eligible null-tenant System accounts. Validated additions and per-added-member audit records now share one SQL Server Serializable transaction. Repeated member IDs and already existing memberships produce no grant-change audit. Self membership and SUPERADMIN groups remain protected.

**Role permissions (RolePermission):** `Roles/{roleId}/permissions` requires a verified System administrator and Master role, prohibits edits to the SUPERADMIN role, and compares persisted permission assignments before/after the legacy replacement command. Each effective addition or removal produces a grant/revoke audit event within the **same Serializable transaction**. Replacements with an unchanged permission set do not produce change events, even though the legacy command may rewrite rows.

Both paths execute inside EF Core's SQL Server retry execution strategy, with live authorization checks **inside** the transaction rather than before it. The stored actor and exact group/role, member/permission, event type, and old/new assignment states support later authorized audit views.

## Verification and deployment boundary

Real SQL Server integration regression tests exercise idempotent membership changes, changed role-permission sets, injected audit INSERT failures rolling back the corresponding grant, and recovery. Build/Unit and SQL Server CI must pass **on the final PR head** before merge. This document must not claim green checks based on stale commits.

This is a partial P0 Issue #11 slice. It does **not** add member removals, bypass-proof bulk/legacy grant mutation capture, privileged bootstrap auditing, cryptographic tamper evidence against DBAs, a read-only Audit API, the complete authenticated 401/403/204 HTTP matrix, or approved Production deployment. Production granular enforcement flags remain OFF. The ledger's SQL UPDATE/DELETE trigger offers append-only behavior for ordinary database DML but is not tamper-evident against DDL-capable principals.

No production rollout or closing Issue #11 is authorized by this PR.
