# Shinera — Secure System Permission CRUD slice

**Date:** 2026-10-23  
**Status:** In progress; not merged, Production granular enforcement OFF  
**Backend:** [PR #14](https://github.com/sholehamed/Shinera-backend/pull/14) `security/platform-permission-crud-20261023`  
**Related:** [P0 Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11), [Tenant write issue #9](https://github.com/sholehamed/Shinera-backend/issues/9), [ADR-003](../architecture/decisions/ADR-003-AUTHORIZATION-AND-PERMISSION-MODEL.md)

## Context

Previously merged PR #13 established an additive System Role/Permission catalog and optional granular Role mutation gates. Permission CRUD still used only `SystemAdministrator` and globally editable Permission definitions include entries used by the live System authorization handler. Mutating `ROLES.CREATE` or `PERMISSIONS.DELETE` through ordinary CRUD would risk altering the authorization model.

## Current implementation, subject to final CI

- All four System Permission write routes (POST, PUT, PATCH and DELETE) go through `PlatformPermissionWriteExecutor`, delegating to the **existing live-validated `AuthorizedPlatformMasterWrite`** gateway. User state and Master-scoped SuperAdmin are checked against the database on every operation; the read and bounded write scopes are disposed after use.
- A failed authorization or attempt to write a reserved security key returns HTTP 403. The route group remains protected by `SystemAdministrator`, independently returning 401 for anonymous users.
- The reserved `ROLES.*` and `PERMISSIONS.*` namespace cannot be forged on create/rename or mutated through ordinary Permission update, delete or state-toggle, even when granular permission enforcement is OFF. Catalog definitions remain managed through reviewed code/seed/migrations.
- `Identity:PlatformPermissions:EnforcePermissionEndpoints`, OFF by default, optionally attaches independent `RequireSystemPermission("permissions", action)` to each Permission mutation. **No** Production flag is activated or implicit grant created.
- PermissionCreate mapping now tolerates a missing PermissionResources collection.
- SQL Server integration test uses the real migrations plus the registered write interceptor to validate an allowed nonreserved permission modification, catalog immutability, role revocation and scope disposal. The disposable SQL Server API smoke also exercises authorized admin attempt to mutate `ROLES.CREATE` (403) and an unauthenticated request (401).

## Non-goals / remaining P0

- The flag is NOT a standalone guarantee of comprehensive least-privilege: management of user/group/role assignments, the original Role mutation flag, dynamic grant creation, complete production grant bootstrap, token replay and raw SQL/bulk paths need independent checks.
- The `Permission` table stores global permission definitions. Tenant-level write ownership is not provided by an `IMustHaveTenant` property and is not implied by this change.
- `PlatformPermissionWriteExecutor` currently verifies the privileged actor at entry. This is not a concurrency-proof authorization transaction; simultaneous revocation requires additional isolation/transaction testing.
- Runtime logs record operation/actor but do not constitute a tamper-evident durable audit trail. No complete authorized POST/PUT/PATCH/DELETE HTTP matrix is claimed.
- A reviewed one-time Production migration must establish correct grants before enabling `EnforcePermissionEndpoints=true` alongside the existing Role enforcement option. Never turn on dev grant bootstrap for Production, and avoid re-granting revoked permissions.
- Full completion of [Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11) awaits these items. No merge without successful latest CI and final review.

## Verification

| Gate | State |
|---|---|
| Backend Build/xUnit | **21/21 passed**, Release Build — [CI 39909529052](https://github.com/sholehamed/Shinera-backend/actions/runs/39909529052) |
| SQL Server integration migrations | **15/15 passed**, real SQL Server — [CI 39909529059](https://github.com/sholehamed/Shinera-backend/actions/runs/39909529059) |
| Real HTTP Permission 401/403 | **Passed**: real login, role smoke, reserved Permission POST 403, anonymous Permission POST 401 — [CI 39909529059](https://github.com/sholehamed/Shinera-backend/actions/runs/39909529059) |
| PR merge | Not performed |

## Verified PR-head and CI fix — 2026-10-23

- Backend PR #14 latest head: `1dca409efef346d8e3799c28b49602c46129889d`. [Backend CI 39909529052](https://github.com/sholehamed/Shinera-backend/actions/runs/39909529052) passed 21/21 xUnit and Release Build. [Real SQL Server CI 39909529059](https://github.com/sholehamed/Shinera-backend/actions/runs/39909529059) passed 15/15 isolated migration/integration tests and actual API smoke including Permission POST 401/403.
- An intermediate SQL Server workflow failed with exit 127 from erroneously doubled Bash `curl` line-continuation backslashes; no SQL Server test failed. The workflow was fixed and the new PR head passed both checks. Intermediate failing workflow is not final evidence.
- Current change remains a **non-merged, default-off** Permission CRUD authorization hardening slice. Not every positive HTTP CRUD, grant mutation or race/revocation test is covered; durable audit and Production grants remain open.
