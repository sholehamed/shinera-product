# Real HTTP stale-SuperAdmin JWT revocation — Issue #11

**Date:** 2026-10-23  
**Status:** Draft PR; latest backend PR-head CI verified (not merged)  
**Backend:** [PR #25](https://github.com/sholehamed/Shinera-backend/pull/25)  
**Tracking:** [Backend Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11)  
**Baseline:** [merged HTTP authorization matrix, PR #24](PLATFORM-AUTHORIZATION-HTTP-E2E-2026-10-23.md)

## Security requirement

A signed OpenIddict access token with the `superadmin` role claim must **not** remain authorized for privileged System APIs after its persisted fixed-Master SuperAdmin UserRole is removed. Token signature and expiration alone are insufficient for sensitive authorization. The existing `ActiveSystemAdministratorHandler` is designed to re-check the **current database** on each privileged request.

The same principle applies to refresh: an existing refresh token must not issue a fresh privileged token after the persisted role snapshot changes. The existing `RefreshGrantValidator` checks current roles and user state when exchanging refresh tokens.

## Disposable end-to-end acceptance sequence

1. Run the real .NET 10 API over a disposable SQL Server 2022 database with explicit Development-only E2E settings and stable `DBNAME=Shinera_E2E_Platform_HTTP`.
2. Obtain a real signed OpenIddict bearer token and refresh token through actual CAPTCHA + password grant for the seeded System administrator.
3. Assert protected System `GET /api/system/Roles/lookup` responds **200**.
4. Invoke the separately compiled, CI-only SQL mutator to remove precisely one direct SuperAdmin `UserRole` for the seeded null-tenant System admin under the fixed Master storage scope.
5. Send the **identical** signed bearer token after the persisted revoke: System Role GET and POST must both return **403**, not 200/201.
6. Exchange the original refresh token: the token endpoint must respond **400 / invalid_grant**, rather than issue another privileged token.

The real HTTP matrix continues to cover previously reviewed Role/Permission, UserRole, GroupRole, GroupMembers and UserPermission actions from PR #24. It uses non-mocked bearer principals.

## Test-only containment

- The mutator is a **separate test executable**, not an application endpoint or a production-registered service.
- Execution requires `ASPNETCORE_ENVIRONMENT=Development`, `Identity__E2E__Enabled=true`, `SHINERA_E2E_DB_MUTATION=stale-jwt-ci-only`, local SQL Server and a database name beginning with `Shinera_E2E_`.
- Mutation is scoped to the known seeded `ADMIN` System subject and fixed Master SuperAdmin association; unexpected target or row-count errors **roll back**.
- Credentials and signed bearer tokens must not be printed to CI logs.
- Direct SQL here is solely a controlled simulation of *already-persisted authorization revocation* and **is not** an approved production grant revocation workflow or evidence of its transactional audit.
- The test establishes immediate loss of privileged **SystemAdministrator endpoint access**, not global JWT rejection on every arbitrary authenticated endpoint.

## Exit criteria and remaining Issue #11

**Verified final PR-head CI:** Backend head `ba1428f7b9ffe9646bf26e258c05af5b88e9859d` passed [Build/Unit CI 40011896961](https://github.com/sholehamed/Shinera-backend/actions/runs/40011896961) and [real SQL Server + HTTP CI 40011896905](https://github.com/sholehamed/Shinera-backend/actions/runs/40011896905): **20/20 real SQL Server regressions**, **54/54 live HTTP checks**, successful CI-only SQL helper build and one persisted seeded SuperAdmin role removal. The same previously valid bearer returned **403** for privileged GET/POST and the refresh flow returned **400 invalid_grant**. **Still required:** final security review, merge decision, and independent post-merge evidence. These CI results are NOT Production verification.

**Still open under Issue #11:** broader real HTTP Permission CRUD / negative error mapping, audit of denied attempts, concurrency/deadlock tests, Production fixed-Master bootstrap/migration/backfill and rollback, plus final ADR-003/acceptance review. Production granular authorization flags remain **OFF**.
