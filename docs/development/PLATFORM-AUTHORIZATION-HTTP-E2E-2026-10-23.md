# Platform Authorization — real HTTP E2E and SQL Server verification

**Date:** 2026-10-23  
**Status:** Draft PR; final-head CI passed — [Backend PR #24](https://github.com/sholehamed/Shinera-backend/pull/24)  
**Tracking:** [Identity P0 Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11)  
**Backend branch:** `security/platform-authorization-http-e2e-20261023`

## Purpose

Close the HTTP-level evidence gap left by SQL Server and service-level security tests for bounded Master Tenant platform administration. The new `tests/http/platform_authorization_e2e.py` script runs against the **real launched .NET 10 Web API** in the disposable SQL Server CI job, using **real signed OpenIddict access tokens** and actual HTTP status codes. It has no third-party Python dependencies, invented tokens or mocked principal/handler shortcuts.

## Matrix (initial slice)

| HTTP principal / operation | Expected |
| --- | --- |
| Anonymous GET /Roles/lookup, POST /Roles | 401 |
| Invalid bearer token | 401 |
| Logged-in System viewer, GET or POST Role | 403 |
| Logged-in Tenant user, GET or POST Role | 403 |
| Tenant with cross-Tenant `X-Tenant-Id` | 403 |
| Live System SuperAdmin, GET Role list | 200 |
| Authorized Role create / persisted read / update | 201 / 200 / 204 |
| Attempt to replace foreign Role permissions | 403 |
| Attempt to create immutable catalog Permission | 403 |
| Valid Role permission replacement | 204 |
| System viewer direct Role grant/revoke (idempotent) | 204 |
| Attempt to grant System Role to Tenant user | 403 |
| Group Role + Group Member grant/revoke (idempotent) | 204 |
| Attempt to revoke Group membership of Tenant user | 403 |
| System user Permission Allow / Deny / Clear (idempotent) | 204 |
| Administrator attempting self-grant | 403 |

The API's results are asserted **exactly**, including persisted Role data returned through the real read route.

**Issue uncovered by the first real HTTP run:** Role creation returned 201, but a System administrator's subsequent GET of that same persisted Master Role returned 500 because the normal tenant query filter hid Master-scoped definitions from null-tenant System principals. Backend PR #24 now explicitly bypasses the **read** filter only for `RoleGetById` rows with `TenantId == fixed Master Tenant`, after the SystemAdministrator authorization boundary. It does not bypass the write interceptor or reveal other tenants' roles. The PR-head SQL Server + HTTP job must reconfirm this fix.

Follow-up HTTP coverage also validates `Roles/lookup` against the fixed Master role-assignment scope, rejects modifications to the reserved SUPERADMIN Role and reserved permission keys, and exercises Role deletion after cleanup. A later HTTP pass uncovered an unexpected GroupRole revoke 403 after normal GroupMember cleanup. The structured HTTP denial log identified `TenantSaveChangesInterceptor` rejecting a **Modified** GroupRole grant. The root cause was `GroupRole : FullAuditableEntity`: `AuditableEntityInterceptor` silently transformed the deletion into a soft-delete modification before the strict grant-write guard ran. It also risked leaving the GroupRole effective and blocking later re-grants behind its unique index.

Backend PR #24 addresses this using a narrow `IHardDeleteAssignment` marker on `GroupRole`. The shared audit interceptor now preserves actual `EntityState.Deleted` for only these authorization-link entities; **all other soft-deletable entities retain existing behavior**. `TenantSaveChangesInterceptor` continues to authorize the exact group+role pair, and the existing SQL `PlatformGrantAuditEntries` records the committed revoke. The HTTP suite explicitly tests delete, idempotent delete, re-grant, and delete again to prevent regression. This does not change the database schema or weaken isolation. CI on the updated head has now passed. The earlier real SQL Server xUnit suite continues to verify audit-row counts, actor identity, rollback when Audit INSERT fails, and denial of unrelated tracked grant mutations.

## Disposable test-only setup

The pre-existing GitHub job `.github/workflows/backend-sqlserver-ci.yml` starts a SQL Server 2022 container and the .NET API in **Development** with `Identity:E2E:Enabled=true`, `Identity:AllowInsecureHttpForE2E=true`, and the explicit opt-in disposable Master role grants. Its HTTP smoke step now invokes the E2E matrix **before stopping the API process**.

Only when `IHostEnvironment.IsDevelopment()` **and** `Identity:E2E:Enabled=true`, seeding creates/reuses:
- `e2e-viewer`, a System user without SUPERADMIN or TenantId
- `e2e-tenant`, a Tenant user in isolated `e2e-http-ci` Tenant
- a Master-scoped E2E-only Group and a non-reserved test Permission

These values are fixtures, not deployment credentials. The test reads its password from `SHINERA_DEVELOPMENT_ADMIN_PASSWORD` in disposable CI; the Python runner does not print bearer tokens or passwords. It must never run against production or a shared database. Production authorization enforcement flags remain OFF; this slice does not change their defaults.

## Scope limits and remaining acceptance criteria

This step does **not** finish Issue #11. The original acceptance criteria also require a complete authenticated HTTP Role/Permission CRUD matrix, stale privileged-token rejection after **database revocation**, durable/centralized rejected-attempt logging, safe Master grant backfill migration, documented bootstrap/rollback controls, and concurrency/deadlock stress tests.

Additional hardening under separate review: tamper-evident off-database Audit sealing (SQL DDL-capable actors can remove the append-only trigger), authorized history viewing and retention. A passed CI is not independent post-merge or Production verification.

**Verified PR-head evidence:** backend commit `c235c776a5d7a72a388c8ac3934106905cf555ff` passed [Build/Unit CI 39940613530](https://github.com/sholehamed/Shinera-backend/actions/runs/39940613530) and [SQL Server + real HTTP CI 39940613458](https://github.com/sholehamed/Shinera-backend/actions/runs/39940613458). The latter ran **20 real SQL Server regression tests** and **50 real OpenIddict HTTP assertions**, including Grant → Revoke → Re-Grant → Revoke of GroupRole and protected SuperAdmin/catalog boundaries. These are final feature-branch CI results, not independent post-merge or Production evidence.

**Gate:** Keep both PRs Draft and [Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11) OPEN pending explicit final review. Production granular authorization remains OFF.
