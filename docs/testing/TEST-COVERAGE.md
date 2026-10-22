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


## Security test work in progress (not part of main baseline)

- Backend security branch: 9 additional policy/CAPTCHA unit tests, totaling **19/19** in the initial security PR CI ([run 39843466153](https://github.com/sholehamed/Shinera-backend/actions/runs/39843466153)).
- Frontend security PR #2: **9/9 full-stack Playwright passed** ([run 39844816502](https://github.com/sholehamed/Shinera-frontend/actions/runs/39844816502)): 5 existing smoke tests plus 4 new negative security cases (anonymous privileged endpoint access, anonymous mutation, malformed CAPTCHA, missing login context). Against unmerged backend feature branch and disposable SQL Server; not yet merged into main.
- **Not covered yet:** positive admin login, permission revocation, OpenIddict refresh/logout lifecycle, production role/bootstrap migration, tenant cross-write integration cases.
