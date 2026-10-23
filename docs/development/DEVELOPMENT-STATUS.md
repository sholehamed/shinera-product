# Shinera — Current Development Status

**Verified:** 2026-10-22  
**Current source:** [Backend](https://github.com/sholehamed/Shinera-backend) · [Frontend](https://github.com/sholehamed/Shinera-frontend) · [Product](https://github.com/sholehamed/shinera-product)

> The [status report dated 2026-10-06](history/DEVELOPMENT-STATUS-2026-10-06.md) concerns an earlier repository generation. Its M0.2–M0.8 and 176-test claims are archived historical evidence; they **must not** be represented as completed capabilities in the newly initialized October 22 repositories.

## Current source and delivered scope
| Repo | Verified present in current repository |
|---|---|
| Product | PRD, backlog, ADRs, project rules, testing and engineering documentation |
| Backend | .NET 10 solution; System/Identity (Tenant/User/Role/Permissions/OpenIddict), FileStorage, Scalar client generator, shared foundation |
| Frontend | Angular 22 application; public landing, signup UI mock, System management pages, admin login UI; Vitest + Playwright |
| Business modules | Branch, BusinessProfile, Services, Staff, CRM and Appointments from earlier milestone history **not present as corresponding current modules** |

**Login:** System Admin login is functional in development per project team, but authentication and permission enforcement remain under correction. This statement is not full token-lifecycle or production security verification.

## First test foundation (merged)
- [Backend PR #1](https://github.com/sholehamed/Shinera-backend/pull/1) squash [9ffc4c7](https://github.com/sholehamed/Shinera-backend/commit/9ffc4c7dbc815e7efcb64e59207ae7f099f3200f): 10 xUnit tests, backend GitHub Actions.
- [Frontend PR #1](https://github.com/sholehamed/Shinera-frontend/pull/1) squash [d833534](https://github.com/sholehamed/Shinera-frontend/commit/d83353447adb785c3b45fda412993c67ec545649): Angular 22/Vitest, Playwright with real .NET + SQL Server, E2E config and setup fixes.
- Main branch [backend CI passed](https://github.com/sholehamed/Shinera-backend/actions/runs/39841510200), [frontend CI passed](https://github.com/sholehamed/Shinera-frontend/actions/runs/39841585176). [Full-stack PR-head smoke](https://github.com/sholehamed/Shinera-frontend/actions/runs/39839196568): 5 passed.
- Detailed record: [Engineering Log — 2026-10-22](ENGINEERING-LOG-2026-10-22.md); [Coverage](../testing/TEST-COVERAGE.md).

## Next milestones
1. **P0 remaining Identity security:** implement fine-grained permissions, token replay/rotation and SQL Server mutation cases, and reviewed production bootstrap ([Issues #4](https://github.com/sholehamed/Shinera-backend/issues/4) and [#5](https://github.com/sholehamed/Shinera-backend/issues/5)). Baseline group authorization and CAPTCHA guard are merged.
2. Expand now-verified System Admin login/refresh/revocation E2E with SQL Server tenant/role mutation and session expiry cases.
3. Backend SQL Server integration suite with WebApplicationFactory, tenant isolation and migrations.
4. CI reproducibility: pinned cross-repo backend commit and npm lockfile, required checks.
5. Implement core Business vertical slices against PRD/backlog with unit/integration/E2E for each story.

## Rules
Actual current repository code is the source of truth for what exists; PRD/backlog/ADRs determine correct behavior. No inherited completion status from old implementation branches. Record each completed PR and its test evidence in `ENGINEERING-LOG-YYYY-MM-DD.md`.


## Identity/System Security baseline merged — 2026-10-22

[Backend PR #3](https://github.com/sholehamed/Shinera-backend/pull/3) ([merge 6d9432f8](https://github.com/sholehamed/Shinera-backend/commit/6d9432f8cd6645b2acb8fce864031f0020483fb8)) and [Frontend PR #2](https://github.com/sholehamed/Shinera-frontend/pull/2) ([merge af772b2](https://github.com/sholehamed/Shinera-frontend/commit/af772b2052f9c36a71099f006dffd50d6e224ced)) are merged to `main`. [Backend main CI](https://github.com/sholehamed/Shinera-backend/actions/runs/39846613412) passed **19 xUnit**; [final full-stack PR E2E](https://github.com/sholehamed/Shinera-frontend/actions/runs/39846647097) passed **9 Playwright** including anonymous 401 on all twelve System management groups and malformed CAPTCHA `invalid_grant`. Angular/Vitest CI on the final PR head passed 14 tests.

**Scope boundary:** This is an initial security baseline, **not complete Identity security**. Granular ADR-003 permissions, positive admin token lifecycle tests, refresh authorization revalidation, production certificates and role bootstrap are outstanding ([Issue #4](https://github.com/sholehamed/Shinera-backend/issues/4), [Issue #5](https://github.com/sholehamed/Shinera-backend/issues/5)). See [Phase 1 record](IDENTITY-SECURITY-PHASE1.md).

## Final review and merged results — 2026-10-22

- [Backend PR #6](https://github.com/sholehamed/Shinera-backend/pull/6) **merged** via [b2f1beb3](https://github.com/sholehamed/Shinera-backend/commit/b2f1beb301ef1e4ae76a64e9351fa8b3c2632d89). [Final Backend CI](https://github.com/sholehamed/Shinera-backend/actions/runs/39852308481): **21/21 xUnit passed**, Release Build succeeded.
- Final review fixed existing seeded OpenIddict clients lacking Revocation permission; the stored application descriptor is preserved and only the missing permission is added.
- Frontend E2E no longer checks out a temporary backend branch; [Full-stack CI](https://github.com/sholehamed/Shinera-frontend/actions/runs/39852565729) with merged backend main passed **13/13 Playwright** including real admin login, refresh, revocation and non-admin 403.
- [Frontend PR #3](https://github.com/sholehamed/Shinera-frontend/pull/3) **merged** via [3e30c547](https://github.com/sholehamed/Shinera-frontend/commit/3e30c547eb9a5dd9cc535ae877a3c56bf533213a). [Final PR-head Frontend CI](https://github.com/sholehamed/Shinera-frontend/actions/runs/39852565679): **16/16 Vitest passed** with Angular build.
- Remaining: SQL Server token/role/tenant mutation and replay/rotation scenarios, full ADR-003 permissions, production key/client provisioning and JWT invalidation outside System admin endpoints. E2E runs on PR/dispatch, not automatically on main push.


## SQL Server Identity authorization foundation merged — 2026-10-23

[Backend PR #7](https://github.com/sholehamed/Shinera-backend/pull/7) merged into main at [ad1bdad1](https://github.com/sholehamed/Shinera-backend/commit/ad1bdad1d73423b75da3c662b696579a9f1e5208). **21/21 xUnit** and **4/4 SQL Server integration** passed on its final reviewed head. Actual SQL Server migrations and isolated databases verified tenant isolation, tenant suspension, role revocation, active authorization, Tenant `Resource.Action` matching, direct/role/group grant evaluation, explicit user denial precedence and Group/GroupRole tenant consistency. The `RequireTenantPermission(resource, action)` extension is a Tenant-scope foundation, **not yet applied to business endpoints**. System fine-grained permissions, Branch/Own, positive HTTP permission policy E2E, feature gating and token replay/rotation remain open. [Full report](IDENTITY-SQLSERVER-AUTHORIZATION-2026-10-23.md).


## Platform System Resource.Action authorization foundation merged — 2026-10-23

[Backend Draft PR #8](https://github.com/sholehamed/Shinera-backend/pull/8) introduces `RequireSystemPermission(resource, action)` and a default-deny SQL Server-backed handler for platform users (`User.TenantId == null`) using a fixed legacy master permission-definition scope. Final PR-head SQL Server integration **6/6** and xUnit **21/21** runs passed. **Backend PR #8 merged** at [`664b8bb9`](https://github.com/sholehamed/Shinera-backend/commit/664b8bb95a3926b292d5ae4703505627a95741c5). The new requirement is not yet applied to existing System management endpoints. Branch/Own scope awaits concrete business membership and resource ownership entities. **P0 write isolation gap** in `TenantSaveChangesInterceptor` is tracked in [issue #9](https://github.com/sholehamed/Shinera-backend/issues/9). [Detailed implementation and rollout](SYSTEM-RESOURCE-PERMISSIONS-2026-10-23.md).


## Tenant Write Isolation baseline merged — 2026-10-23

[Backend PR #10](https://github.com/sholehamed/Shinera-backend/pull/10) was squash-merged at [`2c53fd8a`](https://github.com/sholehamed/Shinera-backend/commit/2c53fd8a75299e76a03dd85ec6fc8d3666bc60a3). `TenantSaveChangesInterceptor` enforces explicit `WritableTenantIds` and tenant-id immutability on synchronous and asynchronous SaveChanges; development database seeding uses an infrastructure-owned, bounded bootstrap bypass. Real SQL Server **10/10** integration tests and **21/21** xUnit passed on final PR head. Real API startup, seeding, System admin login and protected Roles GET passed. This covers scoped EF Core writes, not all SQL write paths or finished System Role CRUD. [Platform Role/Permission write follow-up #11](https://github.com/sholehamed/Shinera-backend/issues/11); [remaining P0 acceptance #9](https://github.com/sholehamed/Shinera-backend/issues/9). [Full report](TENANT-WRITE-ISOLATION-2026-10-23.md).


## Platform Role write foundation merged — 2026-10-23

[Backend PR #12 — Merged](https://github.com/sholehamed/Shinera-backend/pull/12) ([`3c3e2fce`](https://github.com/sholehamed/Shinera-backend/commit/3c3e2fce55b2c92dcfe10221ce1158a174c43dc2)) provides a live-validated, master-scoped executor for System Role CRUD/state/permission replacement, plus explicit 403 refusal and narrowed SuperAdmin membership. **21/21 xUnit**, **13/13 SQL Server integration**, and actual System Role POST 201/invalid-target PUT 403 smoke passed on the final feature-branch head. The changes are now **merged to main**. Full platform Permission catalog CRUD, durable operation audit, grant bootstrap/migration and expanded HTTP regression remain tracked in [Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11). [Technical record](PLATFORM-ROLE-WRITES-2026-10-23.md).


## Merged baseline — staged System Permission Catalog (2026-10-23)

[Backend PR #13](https://github.com/sholehamed/Shinera-backend/pull/13) **merged** at [`c05bda50`](https://github.com/sholehamed/Shinera-backend/commit/c05bda503a785f20a015c1e1aaa4c7df53e8ddf2), adding stable platform Role/Permission Resource.Action definitions, additive catalog-only startup seed, optional Development-only SuperAdmin grants and an OFF-by-default enforcement switch for five System Role mutation endpoints. This does **not** enable granular policies in Production or perform a production grant migration; existing SystemAdministrator remains in place. The Role/Permission CRUD rollout, reviewable production migration, HTTP grant/revocation test matrix and audit remain open under [Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11). [Detailed notes](PLATFORM-PERMISSION-CATALOG-2026-10-23.md).

- Final PR-head checks: [Backend CI 39872147609](https://github.com/sholehamed/Shinera-backend/actions/runs/39872147609) **21/21**, [SQL Server CI 39872147628](https://github.com/sholehamed/Shinera-backend/actions/runs/39872147628) **14/14** plus real Development Role HTTP smoke. Production fine-grained enforcement stays OFF; Permission CRUD and grant-mutation controls must be completed before any Production rollout.


## Merged slice — System Permission CRUD security (2026-10-23)

[Backend PR #14](https://github.com/sholehamed/Shinera-backend/pull/14) adds live-validated, bounded System Permission CRUD, immutable `ROLES.*` / `PERMISSIONS.*` catalog keys and an OFF-by-default fine-grained Permission policy flag. The full System administrator group guard remains. Final PR-head [Backend CI 39910550319](https://github.com/sholehamed/Shinera-backend/actions/runs/39910550319) passed **21/21** and [SQL Server CI 39910550272](https://github.com/sholehamed/Shinera-backend/actions/runs/39910550272) passed **15/15**, including HTTP 401/403; **merged to main at [`638e24c7`](https://github.com/sholehamed/Shinera-backend/commit/638e24c76ed4486c8967d4f119ea9a37a35f9f39); granular enforcement NOT activated in Production**. [Technical review](PLATFORM-PERMISSION-CRUD-2026-10-23.md). Remaining authorization/production rollout [Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11).

## Merged slice — Platform grant mutation hardening (2026-10-23)

[Backend PR #15](https://github.com/sholehamed/Shinera-backend/pull/15) begins the next [Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11) security slice: reserved live `SUPERADMIN` Role identity cannot be forged or modified using ordinary Role CRUD, and System group membership must pass a bounded live authorization check with Master Group and active System-user validation. SQL Server regressions were added for reserved role writes, group scope, duplicate inserts and revoked admin grants. **Merged to main at [`378d6cac`](https://github.com/sholehamed/Shinera-backend/commit/378d6cac11288ef0b6b832cd94a1de53bb1edcd2); this is not the complete Issue #11 authorization audit.** Production granular enforcement and production grant migration remain OFF/unperformed. [Technical log](PLATFORM-GRANT-MUTATIONS-2026-10-23.md).


## Merged slice — Target-scoped platform grant write boundaries (2026-10-23)

[Backend PR #16](https://github.com/sholehamed/Shinera-backend/pull/16) follows merged PR #15 by making a validated platform master write scope insufficient for **unrelated permission-grant mutations**. The Identity `TenantSaveChangesInterceptor` now blocks direct master-scoped `UserRole`, `GroupRole`, and `UserPermission` mutations within ordinary platform requests, while `RolePermission` and `UserGroup` writes require an explicit, disposable target-scoped intent opened by their authorized endpoint executor. New SQL Server regressions probe direct grants and cross-target writes. **Merged to main at [`a80450d1`](https://github.com/sholehamed/Shinera-backend/commit/a80450d1f33c615d0502527c3f972a08e9ef3d8b)** after successful [Build/Unit CI](https://github.com/sholehamed/Shinera-backend/actions/runs/39914356700) and [SQL Server CI](https://github.com/sholehamed/Shinera-backend/actions/runs/39914356804) on the final reviewed PR head. This is a partial P0 Issue #11 hardening slice: actual direct grant-management workflows, durable audit, concurrency safety and Production migrations remain open. [Technical record](PLATFORM-GRANT-BOUNDARIES-2026-10-23.md).


## Merged slice — Direct System UserRole grant/revoke (2026-10-23)

[Backend PR #17](https://github.com/sholehamed/Shinera-backend/pull/17) introduces bounded PUT/DELETE System Users/{id}/roles/{roleId} endpoints for granting and revoking **non-SuperAdmin Master-scoped roles to other System users only**. The live System admin and role/user scopes are rechecked inside one SQL Server serializable transaction; writes require an exact UserRole pair-specific scope, with no user-supplied TenantId. Self-grants, SuperAdmin assignment/revocation, and Tenant or foreign-role targets are explicitly forbidden. Structured operation logs are not a durable audit store. SQL Server tests cover accepted/idempotent operations and denial/revocation. **Merged into main at [`52571097`](https://github.com/sholehamed/Shinera-backend/commit/52571097ea5bee2c19913982fb0d0e5068e9f7da)**, after successful [Build/Unit CI](https://github.com/sholehamed/Shinera-backend/actions/runs/39916928238) and [SQL Server CI](https://github.com/sholehamed/Shinera-backend/actions/runs/39916927938) on the final reviewed PR head `6c1d600a93285f2bce7035750c6a63c0a2873928`. Direct GroupRole/UserPermission workflows, audit, HTTP E2E and Production bootstrap remain open under Issue #11. [Technical record](PLATFORM-DIRECT-ROLE-GRANTS-2026-10-23.md).


## Merged slice — restricted System GroupRole grants (2026-10-23)

[Backend PR #18](https://github.com/sholehamed/Shinera-backend/pull/18) extends the merged direct UserRole security slice with bounded PUT/DELETE Group Role assignment endpoints. The new transactional service validates the live administrator, fixed Master Group and non-SuperAdmin Role, protects privileged groups, rejects actor self-affecting GroupRole mutations, and prevents new role inheritance for groups with invalid System members. Final security review additionally blocked the indirect self-escalation path through a subsequent administrator self-join to GroupMembers. `TenantSaveChangesInterceptor` accepts master GroupRole writes only for the exact validated GroupId+RoleId intent. The final security-review fix and additional SQL Server regression passed on head `4f15f1335f69e68da1dfb190076bc5185a717eae` ([Build/Unit 39920248562](https://github.com/sholehamed/Shinera-backend/actions/runs/39920248562), [SQL Server 39920248548](https://github.com/sholehamed/Shinera-backend/actions/runs/39920248548)). **Merged to main at [`dd57b8d3`](https://github.com/sholehamed/Shinera-backend/commit/dd57b8d3400891747821df670ad0f538c0d62f44).** **This is a merged partial Issue #11 slice; direct UserPermission management, durable audit and Production rollout remain open.** [Technical record](PLATFORM-GROUP-ROLE-GRANTS-2026-10-23.md).


## In progress — System UserPermission overrides (2026-10-23)

[Backend Draft PR #19](https://github.com/sholehamed/Shinera-backend/pull/19) adds explicit Allow/Deny/Clear endpoints for non-SuperAdmin System user permissions only. Master-scoped `UserPermission` changes require live verified SuperAdmin authority, exact `UserId + PermissionId` grant intent, no self or privileged target, and no override of reserved `ROLES.*` or `PERMISSIONS.*` keys. A direct Deny overrides role/group Allow in the existing `SystemPermissionHandler`; removal restores inherited rights. SQL Server `Serializable` transaction protects grant replacement; integration regressions test precedence and rejected cross-scope writes. The work is **Draft** with [Build/Unit CI](https://github.com/sholehamed/Shinera-backend/actions/runs/39921680594) and [real SQL Server CI](https://github.com/sholehamed/Shinera-backend/actions/runs/39921680707) **passed** on final backend PR head `e71a1a19b9538cd5dcbc0a415c6df3e1eeeecb36`; durable audit, HTTP matrix, Production migration and activation remain open under [Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11). [Technical record](PLATFORM-USER-PERMISSION-OVERRIDES-2026-10-23.md).
