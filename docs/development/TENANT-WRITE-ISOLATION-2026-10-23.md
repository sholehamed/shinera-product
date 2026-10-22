# Shinera — Tenant Write Isolation (P0) — 2026-10-23

**Status:** Scoped EF Core write-isolation baseline merged to backend `main`; broader platform write authorization and bypass audits still open.  
**Issue:** [Backend #9](https://github.com/sholehamed/Shinera-backend/issues/9)  
**Backend branch:** `security/tenant-write-isolation-20261023` — [merged PR #10](https://github.com/sholehamed/Shinera-backend/pull/10)  
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

- **Verified before merge:** the final PR-head [Backend CI 39863656917](https://github.com/sholehamed/Shinera-backend/actions/runs/39863656917) and [SQL Server CI 39863656779](https://github.com/sholehamed/Shinera-backend/actions/runs/39863656779) succeeded.
- System admin write flows handling tenant-scoped platform definitions may require explicit trusted operation scopes. Current System endpoints still use coarse `SystemAdministrator` authentication and do **not** automatically open the new write scope. Test their affected create/update/delete paths before enabling a production deployment; an allowed read must not imply an allowed write.
- Trusted platform writes currently cover only fixed master storage scope. Platform operations on actual business tenants (Tenant itself, which is not IMustHaveTenant, plus child entities) need resource-specific authorization and audit rules rather than a global scope bypass.
- Existing relational FK integrity, raw SQL and bulk API code paths are independent from EF SaveChanges interceptors and need separate review.
- OpenIddict itself and seed bootstrap must remain functional. The development admin and OAuth client seeding integration needs verification through CI/E2E.
- This patch does not claim complete ADR-003 Branch/Own scope support, Subscription gating, or SQL-level Row-Level Security.

## Verification ledger

| Check | State | Evidence |
|---|---|---|
| Backend xUnit & Release Build | **21/21 passed** | [Final head run 39863656917](https://github.com/sholehamed/Shinera-backend/actions/runs/39863656917) |
| Real SQL Server migration/write-isolation tests | **10/10 passed** | [Final head run 39863656779](https://github.com/sholehamed/Shinera-backend/actions/runs/39863656779) |
| Startup migrations/seed + real admin login and protected Roles lookup | **Passed** | [SQL Server CI 39863656779](https://github.com/sholehamed/Shinera-backend/actions/runs/39863656779) — limited to startup/login/GET; mutating Role CRUD still requires a dedicated platform write operation |
| Backend merge | **Merged** | [PR #10](https://github.com/sholehamed/Shinera-backend/pull/10), squash [2c53fd8a](https://github.com/sholehamed/Shinera-backend/commit/2c53fd8a75299e76a03dd85ec6fc8d3666bc60a3) |

All results and merge commits are recorded below. Do **not** treat full production tenant isolation or platform Role CRUD as completed.

## Initial CI and compatibility findings — 2026-10-23

- [Backend CI run 39863193504](https://github.com/sholehamed/Shinera-backend/actions/runs/39863193504) passed **21/21 xUnit** on the initial interceptor/test commit.
- [SQL Server integration run 39863193517](https://github.com/sholehamed/Shinera-backend/actions/runs/39863193517) passed **10/10** (6 existing plus 4 new write-isolation regressions).
- Later commit [b4733673](https://github.com/sholehamed/Shinera-backend/commit/b4733673be4874f9a9a5a9b027b27ea78bb6d8ee) added a real API startup + seed + admin login/role lookup smoke step. **Verified passing** on [SQL Server CI 39863656779](https://github.com/sholehamed/Shinera-backend/actions/runs/39863656779), on the final code head.
- Inspection of current `RoleCreateCommandHandler` found that a System role created from the existing command mapping has no explicit master storage `TenantId`. With default-deny writes this will correctly be refused unless a dedicated, authorized platform writer supplies the master scope. The prior implementation also could not assign an ActiveTenantId for a System user. **Do not silently fix this by letting every System request pick arbitrary TenantIds.** This is a follow-up needed before platform Role CRUD production rollout.
- Only DbContexts registered through the normal module dependency-injection/interceptor pipeline receive this EF interceptor. Raw SQL, bulk writes, and manual DbContext construction require independent access reviews.

## Final review and merge — 2026-10-23

- Reviewed the final [backend PR #10](https://github.com/sholehamed/Shinera-backend/pull/10) head `b4733673be4874f9a9a5a9b027b27ea78bb6d8ee`, confirmed clean mergeability and both successful CI gates. No blocking regression was found in the scoped interceptor/bootstrap changes.
- **Backend merged** at [`2c53fd8a`](https://github.com/sholehamed/Shinera-backend/commit/2c53fd8a75299e76a03dd85ec6fc8d3666bc60a3); the new interceptor now applies in the module's normal DI path to both synchronous and asynchronous EF Core SaveChanges.
- Final pre-merge proof: [21/21 xUnit](https://github.com/sholehamed/Shinera-backend/actions/runs/39863656917), [10/10 SQL Server](https://github.com/sholehamed/Shinera-backend/actions/runs/39863656779), plus a passing real API startup, development admin login and protected Roles GET smoke against migrated/seeded SQL Server.
- **Independent remaining P0:** [Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11) tracks missing explicitly authorized/audited System Role/Permission CRUD, which must not be fixed by unrestricted system write access. [Issue #9](https://github.com/sholehamed/Shinera-backend/issues/9) stays open until its full acceptance criteria, including actual privileged platform write operations, are covered.
- Test coverage is specifically EF Core `SaveChanges` with the standard interceptor registration; raw SQL, `ExecuteUpdate`/`ExecuteDelete`, manually constructed DbContexts and authorization failure-to-HTTP status mapping require separate audit. No claim of SQL Server row-level security is made.
