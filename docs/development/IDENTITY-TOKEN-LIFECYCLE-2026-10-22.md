# Shinera — Identity Token Lifecycle & Real System Login

**Date:** 2026-10-22  
**State:** Scoped feature merged in Backend and Frontend; residual security work remains open  
**Architecture references:** ADR-001, ADR-002, ADR-003  
**Tracked P0:** [Backend Issue #4](https://github.com/sholehamed/Shinera-backend/issues/4), [Issue #5](https://github.com/sholehamed/Shinera-backend/issues/5)

## Working branches and PRs

| Repository | Feature branch | Draft PR |
|---|---|---|
| [Backend](https://github.com/sholehamed/Shinera-backend) | `security/identity-token-lifecycle-20261022` | [#6](https://github.com/sholehamed/Shinera-backend/pull/6) |
| [Frontend](https://github.com/sholehamed/Shinera-frontend) | `test/identity-token-lifecycle-20261022` | [#3](https://github.com/sholehamed/Shinera-frontend/pull/3) |
| [Product](https://github.com/sholehamed/shinera-product) | `docs/identity-token-lifecycle-20261022` | This document's PR |

## Source inspection and problem

- The OpenIddict refresh grant previously reissued the stored principal without checking whether user, tenant or roles were still valid. Removing a role or disabling a user could allow further token renewal.
- Authorization checks were based on a signed JWT `superadmin` role claim; until its expiry, the access token retained the role after it was removed in the database.
- Before this branch, the browser's `AuthService.logout()` only removed locally saved tokens. The frontend branch now sends the refresh token to OpenIddict's standard revocation endpoint, clearing local state regardless of the network result.
- Positive System login and authenticated non-admin 403 were not covered by baseline Playwright (previously 9 negative/browser smoke checks).
- A valid CAPTCHA is required for positive login tests, without weakening production CAPTCHA.

## Implemented on feature branches

1. `RefreshGrantValidator` verifies account active/lockout status, security stamp, account type, TenantId and active Tenant, and compares current effective roles and permissions with the token's claims. A mismatch returns `invalid_grant`, requiring new login.
2. `ActiveSystemAdministratorHandler` checks current active system user and effective `superadmin` membership from the database on each privileged System request, stopping use of stale elevated JWT claims after role removal.
3. Registered OpenIddict OAuth token-revocation endpoint `/api/system/Auth/revoke` and its application permission for newly-seeded clients.
4. Added explicitly Development-only E2E CAPTCHA fixture, gated by **both** `Identity:E2E:Enabled` and `Identity:AllowInsecureHttpForE2E`; no public bypass endpoint. Added a non-admin System test user only when the E2E fixture is enabled.
5. Playwright tests exercise real login, token refresh, token revocation, 403 on authenticated non-admin, `userinfo`, protected API, and Angular browser navigation to `/system/dashboard`.
6. Frontend AuthService now invokes `/api/system/Auth/revoke` with the existing refresh token before discarding the local session; `TokenStorageService.clear()` removes all account keys and userinfo refresh includes its bearer token. Added Angular unit tests for logout revocation and missing-token behavior.
7. Added SQLite-backed EF Core regression tests: removing the persisted SuperAdmin role blocks both refresh and an old elevated JWT on privileged System routes; disabling the account rejects refresh. SQLite test model handles its lack of SQL Server auto-generated rowversion.
8. Backend and Frontend work is isolated on branches with CI gates. Frontend E2E temporarily checks out the unmerged backend security branch.

## Scope, deployment risks, and missing tests

- **Do not enable Development/E2E flags in production.** E2E seed identities/passwords are disposable and must never be reused outside isolated test databases.
- Pre-existing seeded OpenIddict client registrations previously lacked revocation permission because the seeder only created missing clients. **Fixed in Backend PR #6**: update only the missing `Revocation` permission while preserving the existing client's other configuration. A separately managed production client/bootstrap model is still outstanding.
- The pre-existing `GET/POST /Auth/logout` OIDC end-session handler remains separate from OAuth refresh-token revocation. The browser `AuthService.logout()` now uses the standard revocation endpoint. Revocation failure cannot be treated as confirmed server-side sign-out, though the local session is cleared.
- OpenIddict does not automatically validate existing JWT token DB records on each ordinary API request; the added handler protects privileged System endpoints, not every generic endpoint. **Revoking a refresh token does not imply immediate invalidation of a previously issued JWT access token** on arbitrary APIs; access token revocation/session policies need a separate design and tests.
- Dynamic per-resource API permission checks remain a separate ADR-003 milestone. Non-admin 403 must be verified end-to-end; the backend handler's passing tests alone do not prove it.
- Account/security-stamp/role revocation after an issued refresh token still needs database mutation integration tests and token rotation/reuse coverage.
- If E2E fails, keep this document updated with exact CI run ID, failure and fix; do not represent WIP as complete.

## Verification log

| Date | Run | Result |
|---|---|---|
| 2026-10-22 | [Backend CI initial attempt](https://github.com/sholehamed/Shinera-backend/actions/runs/39848615734) | Failed: missing `Microsoft.Extensions.Configuration` using in seed contributor; corrected on feature branch |
| 2026-10-22 | [Backend CI after initial fix](https://github.com/sholehamed/Shinera-backend/actions/runs/39848876548) | Build and original xUnit suite successful |
| 2026-10-22 | [Backend CI with SQLite regressions](https://github.com/sholehamed/Shinera-backend/actions/runs/39850361856) | First attempt failed: SQL Server rowversion not generated by SQLite, plus a required PasswordHash field. Test fixture adjusted without changing production entities |
| 2026-10-22 | [Backend CI final branch head](https://github.com/sholehamed/Shinera-backend/actions/runs/39850718533) | **21/21 xUnit passed** (19 previous + 2 SQLite DB mutation scenarios), Release Build successful |
| 2026-10-22 | [Frontend E2E before logout changes](https://github.com/sholehamed/Shinera-frontend/actions/runs/39849080037) | **13/13 Playwright passed**: admin real login, refresh, userinfo, privileged API, viewer 403, standard revocation, browser-to-dashboard and prior 9 security/smoke tests |
| 2026-10-22 | [Frontend CI after logout updates](https://github.com/sholehamed/Shinera-frontend/actions/runs/39849729956) | **16/16 Vitest passed**, including 2 new AuthService logout unit tests |

## Merge gates

- Backend CI and front Angular CI pass on latest commits.
- Real admin login, refresh, revocation and non-admin 403 Playwright checks pass with isolated SQL Server.
- Frontend E2E's backend checkout is changed back to `main` **after** Backend PR #6 merges and rerun against the merged code.
- Maintain separate docs and code PRs; log precise merge commit IDs and final CI links before closing this phase.

## Still pending at next review

- Latest Frontend CI and E2E after corresponding logout client changes; see final GitHub Actions runs before marking complete. Browser logout sends revocation best-effort and clears local state immediately; network failure cannot guarantee server-side revocation.
- SQLite EF Core mutation tests now validate removal of the admin role and account deactivation against `RefreshGrantValidator` and the privilege handler. **SQL Server-specific** mutation/integration tests, tenant-status changes, and OpenIddict refresh rotation/replay handling remain P0.
- CI verification of additive OpenIddict application permission update on already-seeded installations; production client management/bootstrap remains a separate security consideration.

## Final review discovery (2026-10-22)

The security review uncovered a real compatibility issue in `DefaultIdentitySeedContributor.SeedOpenIddictAsync`: a previously existing public client `postman` was not updated with OpenIddict's `Revocation` endpoint permission. This meant the frontend's new server-side logout could be rejected on databases created before this feature. Backend PR #6 was patched to populate the **existing** application descriptor, append just the missing revocation permission, and update the existing object, without overwriting the client's other values. [OpenIddict endpoint permissions](https://documentation.openiddict.com/configuration/application-permissions) require this grant. Verified by the final successful backend CI; the change has now been merged.

## Final review and merged results — 2026-10-22

- [Backend PR #6](https://github.com/sholehamed/Shinera-backend/pull/6) **merged** via [b2f1beb3](https://github.com/sholehamed/Shinera-backend/commit/b2f1beb301ef1e4ae76a64e9351fa8b3c2632d89). [Final Backend CI](https://github.com/sholehamed/Shinera-backend/actions/runs/39852308481): **21/21 xUnit passed**, Release Build succeeded.
- Final review fixed existing seeded OpenIddict clients lacking Revocation permission; the stored application descriptor is preserved and only the missing permission is added.
- Frontend E2E no longer checks out a temporary backend branch; [Full-stack CI](https://github.com/sholehamed/Shinera-frontend/actions/runs/39852565729) with merged backend main passed **13/13 Playwright** including real admin login, refresh, revocation and non-admin 403.
- [Frontend PR #3](https://github.com/sholehamed/Shinera-frontend/pull/3) **merged** via [3e30c547](https://github.com/sholehamed/Shinera-frontend/commit/3e30c547eb9a5dd9cc535ae877a3c56bf533213a). [Final PR-head Frontend CI](https://github.com/sholehamed/Shinera-frontend/actions/runs/39852565679): **16/16 Vitest passed** with Angular build.
- Remaining: SQL Server token/role/tenant mutation and replay/rotation scenarios, full ADR-003 permissions, production key/client provisioning and JWT invalidation outside System admin endpoints. E2E runs on PR/dispatch, not automatically on main push.
