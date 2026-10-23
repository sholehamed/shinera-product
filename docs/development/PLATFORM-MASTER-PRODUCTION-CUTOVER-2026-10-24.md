# Production Master authorization cutover — P0 Issue #11

**Date:** 2026-10-24  
**Backend:** [PR #29](https://github.com/sholehamed/Shinera-backend/pull/29) · [squash commit `1e206094`](https://github.com/sholehamed/Shinera-backend/commit/1e206094e097c17459c1881f0850d376595a0de1)  
**Tracking:** [P0 Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11)  
**Status:** Phase 5A read-only Inventory reviewed and merged. NOT Production backfill or cutover approval.

## Scope and invariants

This is a narrow implementation slice of ADR-003's **separate platform context, default deny, backend authorization and audited decisions**. It is not an implementation of Tenant/Branch/Own/Child permission-assignment scopes.

- System principals have `User.TenantId = NULL`, type `System`, and a validated live identity; they **are not made members of a regular Tenant**.
- Fixed Master Tenant is the *storage scope* for platform Role, UserRole, RolePermission and similar platform grant relationships. No superadmin cross-Tenant write bypass is restored.
- Nine stable platform Permission definitions are defined by `PlatformPermissionCatalog.Definitions`: `ROLES.CREATE`, `ROLES.UPDATE`, `ROLES.DELETE`, `ROLES.CHANGE-STATE`, `ROLES.ASSIGN-PERMISSIONS`, `PERMISSIONS.CREATE`, `PERMISSIONS.UPDATE`, `PERMISSIONS.DELETE`, `PERMISSIONS.CHANGE-STATE`.
- `Identity:PlatformPermissions:EnforceRoleEndpoints` and `EnforcePermissionEndpoints` remain **OFF in Production**. `AllowCodeUniquenessMigration` also remains **OFF until the earlier migration is separately approved**.
- Existing automatic Production seed only ensures stable catalog definitions. The Development-only SuperAdmin/Grant bootstrap is **not reused in Production**.

## Stage 5A — Inventory (implemented in Backend PR #29)

`PlatformProductionGrantInventory.ReadAsync` returns a **read-only** snapshot. It checks the fixed Master Tenant's activity, exactly one active Master SuperAdmin role, at least one *active* live System subject with a Master-scoped role assignment, System users incorrectly holding Tenant IDs, cross-scope Master UserRole, RolePermission, GroupRole, UserGroup and UserPermission relationships, direct user-level deny overrides, missing/deactivated/duplicate platform codes and missing Master SuperAdmin RolePermission grants. Each invalid Master relationship is an explicit blocking counter, even when SQL foreign keys are satisfied.

A failed gate reports `HasBlockingFindings=true`; a passing gate **only means these selected blockers were not detected**, not that rollout is approved or every possible authorization path was assessed.

The separate **read-only** `scripts/sql/platform-master-production-inventory.sql` requires an explicitly supplied fixed Master GUID and prints scoped IDs and grant rows for internal operator review. Do not share these results publicly. It must be checked against the actual deployed catalog revision and target DB. No read-only inventory operation creates a role, user, membership or permission grant.

Real disposable SQL Server regressions cover missing grants, complete approved-scope grants, revoked grants, wrongly bound System accounts, Tenant subjects with Master role assignments, direct deny overrides, all four additional cross-scope Master relationship types, and no mutation by inventory calls.

## Staged Production execution plan — NOT YET AUTOMATED

1. **Inventory and freeze:** confirm target instance, fixed Master GUID, deployed API version, current enforced flags, actual System admins with named reviewers, backup/recovery time, all Role/User/Group grants and system catalog. Run both SQL read-only inventories (including `platform-permission-code-preflight.sql`). Keep existing granular flags OFF.
2. **Review and approve proposal:** separately enumerate exact `RoleId / PermissionId / UserId` pairs and direct deny exceptions. Audit approvals; do **not** grant all Master permissions to every existing System account. Existing disabled permissions and revoked grants must never be silently reactivated.
3. **Safe bootstrap/backfill (future PR):** execute a specifically reviewed, fixed-Master only grant migration, under a serializable/controlled SQL transaction with preconditions and append-only audit in the *same transaction*. Abort with no partial grants on a mismatch, audit failure, drift or concurrency. Never alter `User.TenantId`.
4. **Shadow and smoke checks (future PR):** repeat Inventory, verify independent real signed OpenIddict Role/Permission HTTP allow and deny paths (including stale token revocation), compare against approved approval matrix, inspect denial/audit logs.
5. **Gradual cutover (future decision):** enable only intended `EnforceRoleEndpoints` / `EnforcePermissionEndpoints` flags in a supervised maintenance window, one gate at a time. Never infer readiness solely from a green CI run.
6. **Rollback (future drill):** disable granular flags first; revoke **only** backfill rows created by that approved deployment (using captured before/after IDs and transactional audit). Do not undo unrelated administrator changes, drop history triggers or reset System accounts. An EF migration downgrade or DB restore requires its own verified runbook.

## Explicit unresolved approval gates

- Offline/Production inventory has **not** run against any customer database; no real System user population or grant drift can be asserted.
- Production backfill executor, exact reversible batch manifest, immutable grant-audit linking, partial failure/fault injection, multi-node concurrency, backup/restore and rollback rehearsal remain to implement and test.
- No authenticated HTTP interface is added for bypassing Master security. Readiness reports must not leak secrets or raw identity details.
- Production audit retention, denial volume, long-running locks and final ADR-003 security acceptance remain open.

**Verified final backend PR head `11484663e4e0d41bac0f70881200822cd9bbdfbe`:** [Build/Unit CI 40027251550](https://github.com/sholehamed/Shinera-backend/actions/runs/40027251550) PASS; [real SQL Server + signed OpenIddict HTTP CI 40027251639](https://github.com/sholehamed/Shinera-backend/actions/runs/40027251639) PASS — **28/28 SQL Server tests, 89/89 HTTP checks**, including three read-only inventory regressions. The final-review fix added explicit blocking checks for Master RolePermission, GroupRole, UserGroup and UserPermission cross-Tenant relationships. These are CI tests using disposable SQL Server, **not** an inventory of a real Production installation.

**Backend PR #29 was squash-merged** at [`1e206094`](https://github.com/sholehamed/Shinera-backend/commit/1e206094e097c17459c1881f0850d376595a0de1). The CI evidence is from the final **PR head**, not post-merge SQL/HTTP or a Production installation. **Issue #11 remains OPEN** for approved backfill, cutover, rollback and final ADR-003 review. Production granular enforcement remains OFF.
