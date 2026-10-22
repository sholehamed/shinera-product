# Shinera — Identity Token Lifecycle & Real System Login

**Date:** 2026-10-22  
**State:** Work in progress — no PR merged  
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
7. Backend and Frontend work is isolated on branches with CI gates. Frontend E2E temporarily checks out the unmerged backend security branch.

## Scope, deployment risks, and missing tests

- **Do not enable Development/E2E flags in production.** E2E seed identities/passwords are disposable and must never be reused outside isolated test databases.
- Existing OpenIddict client registrations may need an explicit production-safe migration to receive revocation endpoint permission; the current seed only creates a client when missing.
- The pre-existing `GET/POST /Auth/logout` OIDC end-session handler remains separate from OAuth refresh-token revocation. The browser `AuthService.logout()` now uses the standard revocation endpoint. Revocation failure cannot be treated as confirmed server-side sign-out, though the local session is cleared.
- OpenIddict does not automatically validate existing JWT token DB records on each ordinary API request; the added handler protects privileged System endpoints, not every generic endpoint. **Revoking a refresh token does not imply immediate invalidation of a previously issued JWT access token** on arbitrary APIs; access token revocation/session policies need a separate design and tests.
- Dynamic per-resource API permission checks remain a separate ADR-003 milestone. Non-admin 403 must be verified end-to-end; the backend handler's passing tests alone do not prove it.
- Account/security-stamp/role revocation after an issued refresh token still needs database mutation integration tests and token rotation/reuse coverage.
- If E2E fails, keep this document updated with exact CI run ID, failure and fix; do not represent WIP as complete.

## Verification log

| Date | Run | Result |
|---|---|---|
| 2026-10-22 | [Backend CI initial attempt](https://github.com/sholehamed/Shinera-backend/actions/runs/39848615734) | Failed: missing `Microsoft.Extensions.Configuration` using in seed contributor; corrected on feature branch |
| 2026-10-22 | [Backend CI after fix](https://github.com/sholehamed/Shinera-backend/actions/runs/39848876548) | Build and existing xUnit suite successful |
| 2026-10-22 | [Frontend E2E](https://github.com/sholehamed/Shinera-frontend/actions/runs/39849080037) | **13/13 Playwright passed**: admin real login, refresh, userinfo, privileged API, viewer 403, standard revocation, browser-to-dashboard and prior 9 security/smoke tests |

## Merge gates

- Backend CI and front Angular CI pass on latest commits.
- Real admin login, refresh, revocation and non-admin 403 Playwright checks pass with isolated SQL Server.
- Frontend E2E's backend checkout is changed back to `main` **after** Backend PR #6 merges and rerun against the merged code.
- Maintain separate docs and code PRs; log precise merge commit IDs and final CI links before closing this phase.

## Still pending at next review

- Latest Frontend CI and E2E after corresponding logout client changes; see final GitHub Actions runs before marking complete. Browser logout sends revocation best-effort and clears local state immediately; network failure cannot guarantee server-side revocation.
- Integration scenario mutating SQL Server user/role/tenant **after** token issuance to verify the refresh guard and per-request database authorization revocation. This remains P0 to avoid counting source inspection as executed evidence.
- For existing OpenIddict clients, a supported production migration enabling the standard revocation endpoint permission.
