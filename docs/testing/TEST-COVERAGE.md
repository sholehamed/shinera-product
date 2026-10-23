# Shinera — Automated Test Coverage Inventory

**Verified:** 2026-10-22. These are executed test counts, not line/branch coverage percentages.

| Layer | Current measured state | CI |
|---|---|---|
| .NET xUnit: TenantContext, TenantResolutionMiddleware, System admin policy and CAPTCHA | **19 / 19 passed** | [Backend main](https://github.com/sholehamed/Shinera-backend/actions/runs/39846613412) |
| Angular Vitest: app shell and selected System components (12 spec files) | **14 / 14 passed** | [Frontend main](https://github.com/sholehamed/Shinera-frontend/actions/runs/39847035820) |
| Playwright API: CAPTCHA, auth denial across 12 System groups, tenant mutation and login errors | **6 / 6 passed** | [Final E2E against backend main](https://github.com/sholehamed/Shinera-frontend/actions/runs/39846647097) |
| Playwright browser smoke: System login UI/CAPTCHA, required fields, public landing | **3 / 3 passed** | [Final E2E against backend main](https://github.com/sholehamed/Shinera-frontend/actions/runs/39846647097) |

**Boundaries:** One legacy API smoke case tolerates either 401 or 403; the newer negative security suite enforces exact anonymous 401 on all twelve System management groups. Neither proves authenticated non-admin 403 or granular RBAC. The E2E does *not* submit successful admin credentials, assert token revocation or test tenant mutations. E2E does not currently run on `main` push.

## Priority gaps
| Priority | Missing regression |
|---|---|
| P0 | Complete privileged System endpoint authorization; exact 401/403; valid admin access |
| P0 | CAPTCHA invalid/expired/replayed token refusal |
| P0 | Successful admin login and System dashboard; refresh and Logout |
| P0 | Real System/Tenant boundary and cross-tenant database isolation |
| P1 | WebApplicationFactory + SQL Server migrations, seed idempotency, EF query filters |
| P1 | Business registration, service catalog, staff, schedule, CRM and appointments as implemented |
| P1 | Appointment concurrency and subscription feature gating |
| P2 | RTL/accessibility, responsive and cross-browser tests |


## Security baseline merged into main

- Backend `main`: **19/19 xUnit passed** (10 original + 9 additional policy/CAPTCHA) — [post-merge main CI 39846613412](https://github.com/sholehamed/Shinera-backend/actions/runs/39846613412).
- Frontend merged PR #2: **9/9 full-stack Playwright passed** ([final pre-merge run 39846647097](https://github.com/sholehamed/Shinera-frontend/actions/runs/39846647097)): 5 original smoke tests + 4 negative security cases (anonymous access to all 12 management groups, anonymous tenant mutation, malformed CAPTCHA, missing login context). Uses the merged backend **main** and disposable SQL Server. E2E workflow runs on PR and manual dispatch, not `main` push.
- **Not covered yet:** positive admin login, authenticated non-admin 403, permission revocation, OpenIddict refresh/logout lifecycle, production role/bootstrap migration, tenant cross-write integration cases. [Issue #4](https://github.com/sholehamed/Shinera-backend/issues/4), [Issue #5](https://github.com/sholehamed/Shinera-backend/issues/5).

## Latest merged Identity Token Lifecycle baseline (2026-10-22)

| Test layer | Verified passing | Evidence |
|---|---|---|
| .NET xUnit | **21/21** | [Backend CI](https://github.com/sholehamed/Shinera-backend/actions/runs/39852308481) |
| Angular Vitest | **16/16** | [Frontend CI](https://github.com/sholehamed/Shinera-frontend/actions/runs/39852565679) |
| Playwright real Angular + backend main + SQL Server | **13/13** | [Final PR-head E2E](https://github.com/sholehamed/Shinera-frontend/actions/runs/39852565729) |

This latest baseline supersedes the older table above for current counts and covered flows: real admin login, UserInfo, refresh, OAuth revoke, non-admin 403, browser redirect and previous negative security checks. SQLite-based role-removal and user-deactivation tests are included in backend 21 tests. SQL Server post-issuance mutations, replay/rotation, full permissions and production hardening remain open; E2E does not run on main push.


## SQL Server authorization regression — merged 2026-10-23

- [Backend SQL Server CI 39857222037](https://github.com/sholehamed/Shinera-backend/actions/runs/39857222037): **4/4 passed**, using EF Core migrations against isolated SQL Server 2022 databases (no SQLite fallback).
- [Backend CI 39857222206](https://github.com/sholehamed/Shinera-backend/actions/runs/39857222206): **21/21 xUnit passed**; Release Build successful.
- Regression cases verify stale platform role and disabled user checks, tenant suspension/filter isolation, Tenant-scope `Resource.Action` grants, direct/role/group grants, explicit denied-user override, tenant consistency and cross-Tenant refusal.
- **Not yet covered:** real API-level HTTP enforcement by `RequireTenantPermission`, System/Branch/Own/Child scopes, subscription gating, and token replay/rotation. These test counts are the final PR-head checks before backend merge, not an independent SQL Server main-push CI run.


## Platform System permission foundation — merged 2026-10-23

- [Backend [PR #8](https://github.com/sholehamed/Shinera-backend/pull/8) merged at [`664b8bb9`](https://github.com/sholehamed/Shinera-backend/commit/664b8bb95a3926b292d5ae4703505627a95741c5). Final PR-head runs: [Backend CI 39859081756](https://github.com/sholehamed/Shinera-backend/actions/runs/39859081756) **21/21 passed** and [SQL Server CI 39859082403](https://github.com/sholehamed/Shinera-backend/actions/runs/39859082403) **6/6 passed**, including actual foreign-Tenant role grants.
- System permissions are validated through DB-backed handler tests but **not yet mapped to real HTTP endpoints**. Do not equate these tests with comprehensive platform permission enforcement.
- Remaining: Platform endpoint policy metadata & 401/403, Branch/Own scopes, write-scope interceptor P0 [#9](https://github.com/sholehamed/Shinera-backend/issues/9), and production permission bootstrap.


## Tenant Write Isolation — merged baseline (2026-10-23)

- [Backend PR #10](https://github.com/sholehamed/Shinera-backend/pull/10) merged at [`2c53fd8a`](https://github.com/sholehamed/Shinera-backend/commit/2c53fd8a75299e76a03dd85ec6fc8d3666bc60a3).
- [Backend CI 39863656917](https://github.com/sholehamed/Shinera-backend/actions/runs/39863656917): **21/21 xUnit passed**, Build successful.
- [SQL Server CI 39863656779](https://github.com/sholehamed/Shinera-backend/actions/runs/39863656779): **10/10 SQL Server tests passed** (includes 4 new write isolation cases and previous 6), plus real app migrations/seed/admin login/Roles GET smoke.
- Tests cover normal in-tenant insert/update, cross-tenant add/update/delete, tenant-id reassignment, synchronous/async writes, scoped platform-master writes, startup bootstrap and disposal boundaries.
- **Not covered:** full System Role/Permission CRUD + HTTP authorization under trusted write scope ([#11](https://github.com/sholehamed/Shinera-backend/issues/11)); bulk/raw SQL and manual DbContexts; full P0 acceptance ([#9](https://github.com/sholehamed/Shinera-backend/issues/9)). The SQL Server test result above is the last PR-head run, not a separate main push run.


## Platform Role write authorization — merged 2026-10-23

- [Final Backend CI 39869951318](https://github.com/sholehamed/Shinera-backend/actions/runs/39869951318): **21/21 xUnit passed** and Release Build successful.
- [Final SQL Server integration 39869951465](https://github.com/sholehamed/Shinera-backend/actions/runs/39869951465): **13/13 tests passed**, including Role master writes, role revocation, foreign-role denial, replacement rollback and existing tenant isolation suites.
- Same SQL Server CI completed real API migrations/seed, System admin login, GET Roles, Role POST **201** and invalid role permission PUT **403**. Full authorized PUT/DELETE/PATCH HTTP coverage is not yet proved.
- [Backend PR #12](https://github.com/sholehamed/Shinera-backend/pull/12) merged at [`3c3e2fce`](https://github.com/sholehamed/Shinera-backend/commit/3c3e2fce55b2c92dcfe10221ce1158a174c43dc2); these counts are final PR-head checks (not independent post-merge SQL Server runs). No production permission catalog migration or durable audit is implemented.


## Merged baseline — Platform Permission Catalog and opt-in Role enforcement

- [Backend PR #13](https://github.com/sholehamed/Shinera-backend/pull/13): SQL Server integration test for catalog idempotence, no implicit grant, explicit Development grant, revoked grant persistence, and inactive permission preservation.
- Real Development app smoke opts into granular Role policy with deliberate disposable test bootstrap; existing Role POST and foreign-scope 403 are checked.
- **Latest PR-head checks passed:** [Backend CI 39872147609](https://github.com/sholehamed/Shinera-backend/actions/runs/39872147609) **21/21** and [SQL Server CI 39872147628](https://github.com/sholehamed/Shinera-backend/actions/runs/39872147628) **14/14**, plus API smoke. Production migration, complete granular System/Permission CRUD, HTTP permission revocation and SQL bulk/raw write isolation have not been validated.

- [Backend PR #13](https://github.com/sholehamed/Shinera-backend/pull/13) **merged** at [`c05bda50`](https://github.com/sholehamed/Shinera-backend/commit/c05bda503a785f20a015c1e1aaa4c7df53e8ddf2). Final CI counts above refer to PR-head runs, not a separate post-merge SQL Server test run. Full Production rollout and permission-grant revocation HTTP testing remain open.
