# Persisted denied System API authorization audit — Issue #11

**Date:** 2026-10-23  
**Backend:** [PR #27](https://github.com/sholehamed/Shinera-backend/pull/27) · [squash commit `a7d9c178`](https://github.com/sholehamed/Shinera-backend/commit/a7d9c178c17b57078c1069508990dc3c03dd4f02)  
**Tracking:** [P0 Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11)  
**Status:** Backend security slice reviewed and merged; feature-head CI passed

## Security objective

A privileged API request rejected with HTTP **401** or **403** must produce an independent, durable database audit entry, even though the rejected operation does not open or commit a grant write transaction. This supplements the existing transaction-bound `PlatformGrantAuditEntries` ledger for successful accepted grant changes; it is not a substitute for that ledger.

Both the **SystemAdministrator policy** denials and the **endpoint-level Forbid()** decisions (such as reserved `ROLES.*`/`PERMISSIONS.*` changes) must be observed. This implementation registers one middleware between authentication and authorization, with a response-start callback scoped to endpoints explicitly decorated with the `SystemAdministrator` policy. This avoids audit duplicates and does not intercept Tenant or public endpoints.

## Persistence and privacy

SQL Server table `dbo.PlatformAccessDenialAuditEntries` stores:

| Column | Meaning |
| --- | --- |
| Id | Generated audit entry identifier |
| TenantId | Fixed Master **audit storage scope**; does not turn a System User into a tenant |
| ActorUserId | Verified System JWT subject, or NULL for anonymous/wrong account context |
| HttpMethod | Bounded GET/POST/PUT/PATCH/DELETE/HEAD/OPTIONS |
| RouteTemplate | Registered server route template; not actual URL, IDs, query strings |
| Outcome | `challenge` for HTTP 401 or `forbidden` for HTTP 403 |
| OccurredAtUtc | SQL Server-generated UTC timestamp |

The handler never writes raw authorization headers, JWTs, refresh tokens, cookies, request/response bodies, IP addresses, real request paths/query parameters or SQL exception text. The SQL table has **no EF DbSet**, and a SQL Server trigger blocks UPDATE/DELETE for append-only DML; this is **not cryptographically tamper-evident** against DBA/DDL privileges. Audit writes use a **separate, parameterized SQL connection** with independent autocommit, never the rejected request's DbContext transaction/change tracker. If an audit insert fails, the request remains denied and the response becomes HTTP 503, not an authorization success.

## Real HTTP + SQL Server tests

The CI-only SQL helper is reused under its existing disposable Development database and explicit opt-in guards. After actual signed OpenIddict HTTP calls, it asserts committed audit evidence for four separate cases:

1. Anonymous Permission GET rejected as 401.
2. Unprivileged System user's Permission POST rejected as 403.
3. Authenticated System SuperAdmin's attempt to write a reserved Permission catalog entry rejected as 403 **by the endpoint**.
4. The formerly privileged SuperAdmin's same signed JWT rejected on a protected Role GET as 403 **after** its persisted Master role was removed.

It also attempts to UPDATE an audit row directly in SQL Server, which the append-only trigger must reject. The existing signed-token HTTP matrix (89 assertions before this slice) is preserved.

## Known limits and gates

Denial logging is scoped to SystemAdministrator APIs; authorization failure on arbitrary Tenant/public endpoints, routine validation 400/404/409, and upstream failures before the middleware are not captured by this policy. Potential audit flooding from unauthenticated traffic, write throughput, alerting, TTL/retention, DBA separation and off-database tamper evidence remain operational and security review items. There is intentionally no new Audit Reader endpoint or default production granular permissions.

**Verified PR-head CI:** backend head `f4e8456aed94ab9a48bf94e3fda20b2bc3bb8638` passed [Build/Unit CI 40019987231](https://github.com/sholehamed/Shinera-backend/actions/runs/40019987231) and [real SQL Server + HTTP CI 40019987299](https://github.com/sholehamed/Shinera-backend/actions/runs/40019987299): **20/20 SQL Server regressions**, **89/89 real signed OpenIddict HTTP status assertions**, and a CI-only SQL read proving four committed denied-access audit categories plus UPDATE trigger rejection. An intermediate implementation used a fresh SqlConnection created from a previously opened EF DbConnection.ConnectionString; this caused a fail-closed HTTP 503 regression during denial. The final implementation obtains an **unopened DI-registered connection factory** instead; both latest-head CI workflows passed. No sensitive connection details are logged.

**Review and merge:** final code/security review completed; [Backend PR #27](https://github.com/sholehamed/Shinera-backend/pull/27) was squash-merged at [`a7d9c178`](https://github.com/sholehamed/Shinera-backend/commit/a7d9c178c17b57078c1069508990dc3c03dd4f02). The reported SQL Server and HTTP successes remain **feature-head evidence**, not independent post-merge SQL Server or Production verification. Known operational restrictions remain open. [Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11) remains open for concurrency/rollback, safe Production fixed-Master grant cutover, ADR-003 acceptance and remaining audit hardening.
