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


## Platform System permission branch — in progress, not main

- [Backend SQL Server CI 39858625169](https://github.com/sholehamed/Shinera-backend/actions/runs/39858625169): **6/6 integration** passed at initial PR head (4 previous + 2 new platform permissions scenarios); [xUnit CI 39858625068](https://github.com/sholehamed/Shinera-backend/actions/runs/39858625068): **21/21**. Latest commit also confirmed green: [Backend CI 39859081756](https://github.com/sholehamed/Shinera-backend/actions/runs/39859081756) **21/21 passed** and [SQL Server CI 39859082403](https://github.com/sholehamed/Shinera-backend/actions/runs/39859082403) **6/6 passed**, including actual foreign-Tenant role grants.
- System permissions are validated through DB-backed handler tests but **not yet mapped to real HTTP endpoints**. Do not equate these tests with comprehensive platform permission enforcement.
- Remaining: Platform endpoint policy metadata & 401/403, Branch/Own scopes, write-scope interceptor P0 [#9](https://github.com/sholehamed/Shinera-backend/issues/9), and production permission bootstrap.
