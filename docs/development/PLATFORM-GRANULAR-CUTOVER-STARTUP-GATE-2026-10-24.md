# P0 Issue #11 — Stage 5C: Production granular authorization startup interlock

**Date:** 2026-10-24  
**Status:** Draft implementation slice. **Not a Production deployment or cutover approval.**  
**Code:** [Backend Stage 5C branch](https://github.com/sholehamed/Shinera-backend/tree/security/platform-granular-cutover-startup-gate-20261024)  
**Earlier stages:** [5A inventory](PLATFORM-MASTER-PRODUCTION-CUTOVER-2026-10-24.md), [5B offline backfill/rollback](PLATFORM-MASTER-BACKFILL-ROLLBACK-2026-10-24.md)

## What changed

The existing minimal API already reads `Identity:PlatformPermissions:EnforceRoleEndpoints` and `EnforcePermissionEndpoints` once during endpoint registration; **both default OFF in Production**. Stage 5C adds a **read-only fail-closed check at application startup**, after the existing Identity seeder and *before* endpoint registration. It applies **only** when the environment is Production and **at least one** granular Role or Permission endpoint flag is enabled.

Before accepting an enabled flag, `PlatformGranularCutoverStartupGuard` requires **all** of:

1. Independent default-OFF deployment interlock `Identity:PlatformPermissions:AllowProductionGranularCutover=true`.
2. `Identity:PlatformPermissions:ReviewedMasterTenantId` exactly equals the fixed compiled `DefaultSeedData.MasterTenant.Id`.
3. `Identity:PlatformPermissions:ReviewedDatabaseName` equals the *actual* SQL Server connection database name.
4. Earlier `20261024200000_permissioncodeuniqueness` migration already applied. Its old, independently reviewed Production opt-in / historical data cleanup is still required.
5. No blockers from **read-only** `PlatformProductionGrantInventory.ReadAsync`, including missing/inactive/duplicate catalog codes, missing Master SuperAdmin role grants, direct user denies, invalid cross-Tenant assignments and inactive platform storage.

If any condition fails, **Startup throws before registering platform endpoints** rather than silently treating the two feature flags as a complete Production rollout authorization. This does not update flags, seed grants, touch `User.TenantId`, run a missing uniqueness migration, or modify existing authorization paths. Development/CI retains its opt-in test behavior.

## Operator sequence — NOT automated / not authorized for Production

1. Keep granular flags OFF. Conduct approved database backups and restore drill, collect a signed change ticket and identify the exact reviewed instance, fixed Master ID and deployed commit; freeze concurrent grant mutations.
2. Run both read-only SQL preflight scripts and a reviewed inventory. Resolve anomalies through separately approved remediation. Confirm active operator and reviewer and individually review every expected Role, Group and direct User permission. Confirm the 5B batch manifest/receipt outside application code. No automatic grant-all action.
3. Separately approve and install the historical Permission Code normalized uniqueness migration using its own safe opt-in and database-maintenance procedure; **do not** set `AllowCodeUniquenessMigration` merely to bypass a failing cutover.
4. Independently authorize reviewed Master RolePermissions using the controlled 5B offline process **only when cryptographic approval and exact batch provenance/rollback controls are ready**. Those Production controls remain unimplemented.
5. During an approved maintenance window, provide the fixed reviewed database and Master identifiers, set `AllowProductionGranularCutover=true` and enable **one** named granular endpoint family at a time. Restart **every** application process; require successful startup/read-only gate checks.
6. Use real signed OpenIddict System actor tokens to verify allow/deny, stale-token revocation, a role without the granular grant and rejection of Tenant subjects. Monitor persisted authorization failures, grant-audit integrity, health and lock pressure. Continue only after manual approval.
7. **Emergency rollback of enforcement:** set `EnforceRoleEndpoints=false` and `EnforcePermissionEndpoints=false` in effective configuration for **every instance**; restart/roll all instances and verify access against live coarse `SystemAdministrator` policy. The guard bypasses all new DB checks in OFF mode; it **does not** add, delete or reactivate grants. Revoke exact 5B grants only with a separate reviewed, correctly attributed manifest and recovery authorization. Do not drop the uniqueness index merely to turn granular checks OFF.

## Critical limitations

- `AllowProductionGranularCutover`, reviewed DB and Master IDs are **configuration acknowledgements**. They do **not** cryptographically verify an actual independent signed operator approval or change ticket, and cannot by themselves constitute production acceptance.
- Read-only inventory is a **startup snapshot**, not a live multi-instance distributed cutover coordinator or tenant-wide effective permission proof. Existing System and granular handlers still perform current database lookups per request; SQL data or configuration may change after startup.
- Endpoint flags are resolved at **registration**, so editing config without restarting a process does not reliably turn an already registered permission policy on/off.
- A clean inventory reflects a narrow superadmin path: it cannot prove that all customer deployments, administrators, branch/own/child scope rules, non-platform modules or planned access are correct. Retain human approval and shadow verification.
- 5B remains an **unwired offline primitive**, not a production-ready signed deployment tool. Full rollback rehearsal, immutable batch audit provenance, dual approval, recovery for ambiguous timeouts and ADR-003 final acceptance remain open.
- If a Production deployment already has granular flags enabled without the new interlock values, the new version will deliberately **refuse startup**; rollout must explicitly account for and approve this breaking deployment safeguard. It should not be bypassed by setting everything to true automatically.

## Disposable SQL Server acceptance matrix

- Both granular flags OFF: Production preflight permits startup without extra database reads or grants, including after a missing/revoked permission.
- Development opt-in: existing disposable CI behavior unaffected.
- Either Role or Permission granular flag ON in Production without acknowledgement: denied.
- Wrong fixed Master tenant or reviewed DB name: denied.
- Correct acknowledgement with incomplete grants: denied.
- Correct acknowledgement, previously applied uniqueness migration, complete verified Master grants: preflight succeeds without writes.
- Revoking one of the reviewed Master grants: next Production preflight fails; changing only both flags to OFF permits safe return to coarse enforcement without restoring the grant.
- Legacy database without uniqueness index: denied despite acknowledgement.
- Existing real OpenIddict HTTP endpoint authorization and denial audit matrix must remain green.

## Verified PR-head test evidence

[Backend PR #31](https://github.com/sholehamed/Shinera-backend/pull/31), head `4a941c2f5b15feaa24e42d3c79eb2c3dcf1ebed8`, passed [Build/Unit CI 40086712581](https://github.com/sholehamed/Shinera-backend/actions/runs/40086712581) and [real SQL Server + signed HTTP CI 40086712579](https://github.com/sholehamed/Shinera-backend/actions/runs/40086712579): **36/36 real disposable SQL Server tests** (three new cutover gate cases), **89/89 signed OpenIddict HTTP assertions** and persisted denial audit checks. These are not a Production simulation or proof of signed deployment authorization. Both PRs remain Draft and unmerged.

**No Production configuration was changed. Issue #11 stays OPEN until a supervised live change-control rehearsal, independent signed approvals, real multi-instance rollback and final ADR-003 acceptance.**
