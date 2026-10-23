# Shinera — Platform Role/Group grant mutation hardening

**Date:** 2026-10-23  
**Status:** In progress, Draft — not merged or enabled in Production  
**Backend:** [PR #15](https://github.com/sholehamed/Shinera-backend/pull/15) (`security/platform-grant-mutation-hardening-20261023`)  
**Tracking:** [P0 Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11), [ADR-003](../architecture/decisions/ADR-003-AUTHORIZATION-AND-PERMISSION-MODEL.md)

## Threat model and selected first slice

The existing live SystemAdministrator check depends on an active master-scoped role whose normalized name is `SUPERADMIN`. Creating, renaming, deleting or deactivating this role through generic Role CRUD can forge or disable privileged authorization. Separately, the legacy Group member command accepted arbitrary IDs with no master/UserType validation and no tenant storage assignment. Existing tenant write interception prevents unscoped writes but is not an adequate resource-level grant policy.

## Backend changes under review

1. Platform Role writes now query the **live master role definition** inside the bounded `AuthorizedPlatformMasterWrite` scope. Generic Role create/update cannot assign the reserved `SUPERADMIN` name. Generic CRUD/state/permission replacement cannot modify the existing role with this privileged name.
2. New `PlatformGroupMembershipWriteExecutor` uses the existing live account/master-role authorization scope and verifies group membership inputs against SQL Server before dispatch. It accepts only active master-scoped groups and active System users with `User.TenantId == null`. A group already linked to a `SUPERADMIN` role cannot gain members through ordinary CRUD.
3. `AddGroupMembersCommand` receives a server-owned, JSON-ignored storage tenant. It independently checks group and member scope and writes master-scoped `UserGroup` rows, avoiding duplicate membership insertions.
4. SQL Server regression cases exercise permitted group member addition, duplicate idempotence, denied tenant/foreign/privileged group targets, live superadmin role revocation, reserved Role create/rename and denied reserved Role mutation.

## Verification

- [Backend PR #15](https://github.com/sholehamed/Shinera-backend/pull/15) remains **Draft**.
- [Build/xUnit workflow](https://github.com/sholehamed/Shinera-backend/actions/runs/39912421977): check actual latest-head result before merge.
- [SQL Server authorization workflow](https://github.com/sholehamed/Shinera-backend/actions/runs/39912421998): check actual latest-head result before merge.
- These workflows run on the **feature branch**, not on merged `main`. Do not claim final release verification.

## Known limitations and remaining P0 scope

- System Users and Groups create/update/delete/state APIs remain legacy surfaces requiring independent live-target authorization, trust boundaries and comprehensive HTTP matrix. This slice only gates **group member assignment** and **privileged Role identities**.
- `UserRole`, `GroupRole`, `UserPermission` and other grant writers must be inventoried and guarded with verified actor/target scope, including direct command use, group-based permission inheritance and self-escalation.
- Role rename/protection checks and group membership checks are not concurrency-proof by themselves; use transaction/isolation guarantees where needed. No durable tamper-evident audit has been implemented; structured logs are only preliminary audit signals.
- A reviewed Production SuperAdmin bootstrap, granular policy enablement and migration path are **not part of this PR**. Do not enable default-off enforcement or seed privileged grants.
- Preserve existing `TenantSaveChangesInterceptor` fail-closed behavior. Never give all System users unrestricted Tenant write access.

**Exit criteria for P0 Issue #11** remain broader than this incremental security slice.
