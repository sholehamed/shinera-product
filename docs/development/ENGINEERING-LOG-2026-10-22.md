# Shinera — Engineering Log — 2026-10-22

**Result:** Completed and merged  
**Area:** Identity/System testing foundation, Angular 22 + .NET 10 + real SQL Server CI  
**Canonical home:** `shinera-product/docs/development/`

## Delivery

| Repository | Merged PR | Squash commit | Changes |
|---|---|---|---|
| Backend | [#1](https://github.com/sholehamed/Shinera-backend/pull/1) | [9ffc4c7](https://github.com/sholehamed/Shinera-backend/commit/9ffc4c7dbc815e7efcb64e59207ae7f099f3200f) | xUnit project, tests for TenantContext and TenantResolutionMiddleware, backend CI |
| Frontend | [#1](https://github.com/sholehamed/Shinera-frontend/pull/1) | [d833534](https://github.com/sholehamed/Shinera-frontend/commit/d83353447adb785c3b45fda412993c67ec545649) | Angular/Vitest CI, Playwright browser + API, SQL Server and real .NET orchestration, E2E config, test setup repairs, HttpClient registration |

Both PRs were reviewed and squash-merged to their respective `main` branches on 2026-10-22. No business capability is claimed complete by these testing changes.

## Confirmed verification

| Gate | Result | Evidence |
|---|---|---|
| Backend PR Build/xUnit | 10 passed | [39838572366](https://github.com/sholehamed/Shinera-backend/actions/runs/39838572366) |
| Frontend PR Build/Vitest | 14 passed across 12 files | [39839196557](https://github.com/sholehamed/Shinera-frontend/actions/runs/39839196557) |
| Full-stack E2E latest PR head | 5 passed, Angular + ASP.NET Core + disposable SQL Server | [39839196568](https://github.com/sholehamed/Shinera-frontend/actions/runs/39839196568) |
| Backend `main` after merge | Success | [39841510200](https://github.com/sholehamed/Shinera-backend/actions/runs/39841510200) |
| Frontend `main` after merge | Success | [39841585176](https://github.com/sholehamed/Shinera-frontend/actions/runs/39841585176) |

**Evidence boundary:** E2E currently runs on PR / manual dispatch, not push to `main`. The verified 5-test suite was the PR-head run and is not proof of a separate post-merge E2E run. It uses the backend `main` at checkout time rather than pinning an immutable backend commit.

## Issues found during test foundation

- Angular 22.2 peer dependencies conflicted with TypeScript 7.0; aligned to TypeScript 6.0.3.
- Angular starter test was obsolete; updated for actual router-outlet shell.
- Three component specs lacked required TestBed providers (Material dialog data and route/service dependencies); repaired and expanded assertions.
- `provideHttpClient()` was not configured globally; added.
- Backend compiles with numerous warnings; a warning audit is future work.
- The baseline Identity module still contains P0 security gaps (permission enforcement disabled/incomplete, CAPTCHA Boolean not enforced in login). Tracked at [Backend Issue #2](https://github.com/sholehamed/Shinera-backend/issues/2). Green smoke tests do not certify authorization security.

## Next sequence

1. Correct P0 authorization and CAPTCHA enforcement with negative API/integration coverage.
2. Test full admin Login → System dashboard flow and token refresh/Logout with isolated test identities.
3. Add WebApplicationFactory + isolated SQL Server integration tests (EF migrations/filters, auth).
4. Add a frontend lockfile and `npm ci`; pin backend revision or contract for cross-repository E2E reproducibility.
5. Require CI checks before merging PRs and expand test coverage as product slices are built.

## Engineering log policy

For each significant development PR, record: date, branch, PR, merge commit, changed area, product story/ADR link, verification/CI URLs, outstanding debt and security implications. Append to this log for same-day changes and create a new `ENGINEERING-LOG-YYYY-MM-DD.md` for later days. Keep `DEVELOPMENT-STATUS.md` and `docs/testing/TEST-COVERAGE.md` aligned with the current source.


---

## Identity/System Phase 1 started — 2026-10-22

- Backend feature branch `security/identity-system-hardening-20261022`, [PR #3](https://github.com/sholehamed/Shinera-backend/pull/3).
- 12 System management endpoint groups now require a dedicated System Administrator policy: authenticated `system` context, no `tenant_id`, explicit `superadmin` role. Public Tenant resolve and login/CAPTCHA remain explicitly anonymous.
- Login password grant now enforces CAPTCHA result and fails closed on unsupported/missing login context.
- Development-only seed ensures an administrator role assignment; no known-password System admin is seeded in Production.
- Added policy/CAPTCHA unit tests; first [backend CI run](https://github.com/sholehamed/Shinera-backend/actions/runs/39843466153) **passed all 19 unit tests**.
- Frontend branch `test/identity-system-security-20261022`, [PR #2](https://github.com/sholehamed/Shinera-frontend/pull/2): Playwright negative auth/CAPTCHA tests against temporary backend feature branch checkout. Status of latest frontend security E2E must be checked before completion.
- Detailed implementation and remaining risks: [Identity Phase 1](IDENTITY-SECURITY-PHASE1.md). Follow-up production role mapping, dynamic permissions, refresh/revocation and stronger integration tests remain open.
