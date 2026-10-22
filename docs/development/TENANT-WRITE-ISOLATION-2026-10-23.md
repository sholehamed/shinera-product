# Shinera — Tenant Write Isolation (P0) — 2026-10-23

**Status:** Work in progress, not merged.  
**Issue:** [Backend #9](https://github.com/sholehamed/Shinera-backend/issues/9)  
**Backend branch:** `security/tenant-write-isolation-20261023` — [Draft PR #10](https://github.com/sholehamed/Shinera-backend/pull/10)  
**Product docs branch:** `docs/tenant-write-isolation-20261023`

## Initial findings from current code

1. In `TenantSaveChangesInterceptor`, the checks for membership in `WritableTenantIds` and `TenantId` reassignment had been commented out.
2. `tenant.IsFilterDisabled` also short-circuited write authorization: granting an internal read filter override inadvertently allowed arbitrary writes.
3. Only `SavingChangesAsync` was intercepted, not synchronous `SavingChanges`.
4. System users intentionally have no TenantId, and `TenantContext.InitializeSystem()` gives no implicit writable tenants. The legacy fixed Master Tenant Id is used to **store** platform roles/permissions, not as the System user's home tenant.
5. Startup seeding runs without a logged-in user; simply re-enabling tenant write checks without an explicit seed scope would break development admin-role and OpenIddict seeding.

## Work implemented in Backend PR #10

- `TenantSaveChangesInterceptor` now denies changes to `IMustHaveTenant` unless there is a current authenticated Tenant context with the target id in `WritableTenantIds`, or an explicitly opened trusted infrastructure scope.
- Assigns an unset TenantId on Added rows only from the current active writable tenant.
- For Modified and Deleted tracked entries, refuses TenantId reassignment. Guards both `SaveChanges()` and `SaveChangesAsync()`.
- Query-filter read override **never** authorizes writes.
- `TrustedTenantWriteScope` has infrastructure-internal entry points for startup bootstrap and explicitly verified System SuperAdmin operations *bounded to* the legacy master storage tenant. It records activation/deactivation in logs; it cannot be opened merely via a user-supplied TenantId. No HTTP endpoint opens this scope automatically.
- `DbSeeder.SeedAsync` explicitly opens and disposes the scoped bootstrap privilege around migrations/seeding.
- Four new SQL Server integration cases verify allowed in-tenant writes and default TenantId, blocked foreign insert/update/delete/sync SaveChanges, prevented reassignment, System/anonymous default denial, bounded explicit platform scope, and non-leaking bootstrap scope.

## Design limits / review gates

- **Do not merge while the CI is unverified.** Check latest Backend CI and isolated SQL Server job, resolve any failures within the new branch.
- System admin write flows handling tenant-scoped platform definitions may require explicit trusted operation scopes. Current System endpoints still use coarse `SystemAdministrator` authentication and do **not** automatically open the new write scope. Test their affected create/update/delete paths before enabling a production deployment; an allowed read must not imply an allowed write.
- Trusted platform writes currently cover only fixed master storage scope. Platform operations on actual business tenants (Tenant itself, which is not IMustHaveTenant, plus child entities) need resource-specific authorization and audit rules rather than a global scope bypass.
- Existing relational FK integrity, raw SQL and bulk API code paths are independent from EF SaveChanges interceptors and need separate review.
- OpenIddict itself and seed bootstrap must remain functional. The development admin and OAuth client seeding integration needs verification through CI/E2E.
- This patch does not claim complete ADR-003 Branch/Own scope support, Subscription gating, or SQL-level Row-Level Security.

## Verification ledger

| Check | State | Evidence |
|---|---|---|
| Backend xUnit & Release Build | In progress | [PR #10 checks](https://github.com/sholehamed/Shinera-backend/pull/10/checks) |
| Real SQL Server migration/write-isolation tests | In progress | [PR #10 checks](https://github.com/sholehamed/Shinera-backend/pull/10/checks) |
| Startup/API compatibility with System role management | Not yet verified | Further integration tests required |
| Main branch merge | Not performed | Await review |

All results, fixes, risk decisions, PR and merge commit hashes must be added to this document and dated engineering log before treating this phase as done.
