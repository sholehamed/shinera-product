# Shinera — Automated Test Coverage Inventory

**Verified:** 2026-10-22. These are executed test counts, not line/branch coverage percentages.

| Layer | Current measured state | CI |
|---|---|---|
| .NET xUnit: TenantContext and TenantResolutionMiddleware | **10 / 10 passed** | [Backend main](https://github.com/sholehamed/Shinera-backend/actions/runs/39841510200) |
| Angular Vitest: app shell and selected System components (12 spec files) | **14 / 14 passed** | [Frontend main](https://github.com/sholehamed/Shinera-frontend/actions/runs/39841585176) |
| Playwright API smoke: anonymous CAPTCHA and role lookup authentication | **2 / 2 passed** | [PR-head E2E](https://github.com/sholehamed/Shinera-frontend/actions/runs/39839196568) |
| Playwright browser smoke: System login UI/CAPTCHA, required fields, public landing | **3 / 3 passed** | [PR-head E2E](https://github.com/sholehamed/Shinera-frontend/actions/runs/39839196568) |

**Boundaries:** The API smoke currently permits either 401 or 403 for role lookup; it does not prove full RBAC. The E2E does *not* submit successful admin credentials, assert token revocation or test tenant mutations. E2E does not currently run on `main` push.

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
