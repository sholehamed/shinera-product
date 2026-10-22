# Engineering Log — Platform Role Writes — 2026-10-23

- Created Backend branch `security/platform-role-permission-writes-20261023` and Product branch `docs/platform-role-permission-writes-20261023` from verified main.
- Reviewed issue #11, TenantWriteIsolation's trusted scope, System authorization and Role commands.
- Implemented request-scoped master Role writer with live actor, role and storage scope checks, operation logs and bounded temporary read/write scopes.
- Routed Role create, update, status, delete and permission replacement through the writer.
- Fixed RoleCreate mapping and added server-owned master scope to created Role/RolePermission.
- Added SQL Server regressions for valid write, foreign scope/role attempt, role revocation, RolePermission replacement and invalid permission atomicity.
- Initial Backend CI failed on attribute namespace resolution; corrected to `global::System.Text.Json.Serialization.JsonIgnore`. Latest checks awaiting confirmation.
- **No code merged.** Fine-grained permission catalog, real HTTP status tests, production migration and full permission CRUD remain open.

See [Phase engineering record](PLATFORM-ROLE-WRITES-2026-10-23.md).

## Final verified branch CI results (not merged)

- Updated all System Role writes to return explicit 403 when the trusted scope or live authorization check fails.
- Extended SQL Server workflow to perform actual System admin RoleCreate POST, expecting HTTP 201, and invalid target RolePermission PUT expecting HTTP 403. A real foreign-Tenant Role is rejected in the isolated SQL Server executor tests.
- Restricted `ActiveSystemAdministratorHandler` role assignments to both master-scoped UserRole and master-scoped Role, fixing possible cross-Tenant role-name spoofing.
- [Backend CI 39868009455](https://github.com/sholehamed/Shinera-backend/actions/runs/39868009455): **21/21 passed** and Build successful.
- [SQL Server CI 39868009445](https://github.com/sholehamed/Shinera-backend/actions/runs/39868009445): **12/12 integration tests passed**; real HTTP API startup/login, Role POST 201 and out-of-scope RolePermissions PUT 403 also passed.
- Backend PR #12 and Product PR #8 remain **Draft; not merged**. Issue #11 remains open for complete permission catalog CRUD, durable audit, production bootstrap and expanded API tests.

- Added independently testable `AuthorizedPlatformMasterWrite` gateway and a SQL Server post-issuance role-revocation/scope-lifetime test. [Backend CI 39869241494](https://github.com/sholehamed/Shinera-backend/actions/runs/39869241494) passed **21/21**; [SQL Server CI 39869241918](https://github.com/sholehamed/Shinera-backend/actions/runs/39869241918) passed **13/13** and live HTTP role-create/forbidden-write smoke.
- Review note: Role API endpoints still use `PlatformRoleWriteExecutor`, which duplicates parts of the new gateway's live checks. The duplicate validation was later consolidated. Broader CRUD/fine-grained permission tests remain outside this PR.


## Final review: single verified write gateway

- Consolidated duplicated live-account, master-role and tenant checks in **one** `AuthorizedPlatformMasterWrite.BeginAsync` gateway. `PlatformRoleWriteExecutor` now delegates to this gateway and handles only operation execution and logging.
- Validates optional target Role ownership in the fixed Master Tenant **before** privileged read/write scopes are opened; missing/foreign role ids fail closed.
- Existing SQL Server regression constructors were updated to invoke the real executor through this gateway. All Role CRUD HTTP routes remain protected by `SystemAdministrator` as well.
- The refactor is isolated in [backend commit cb6e4eb3](https://github.com/sholehamed/Shinera-backend/commit/cb6e4eb3e83cd367a3efff94df97c3c8472d6246). Latest CI [Backend](https://github.com/sholehamed/Shinera-backend/actions/runs/39869951318) and [SQL Server](https://github.com/sholehamed/Shinera-backend/actions/runs/39869951465) were verified successful on the final backend head: **21/21 xUnit** and **13/13 SQL Server**.

## Final security review and merge

- Removed duplicated platform-account and scope checks by making `AuthorizedPlatformMasterWrite` the single live authorization and privileged scope gateway; Role executor now only runs/logs operations.
- Reviewed and passed [Backend CI 39869951318](https://github.com/sholehamed/Shinera-backend/actions/runs/39869951318) **21/21** and [SQL Server CI 39869951465](https://github.com/sholehamed/Shinera-backend/actions/runs/39869951465) **13/13**, including real API login/Role create 201 and cross-scope permission PUT 403.
- [Backend PR #12](https://github.com/sholehamed/Shinera-backend/pull/12) **squash merged** to main at [`3c3e2fce`](https://github.com/sholehamed/Shinera-backend/commit/3c3e2fce55b2c92dcfe10221ce1158a174c43dc2), source head `cb6e4eb3`.
- Issue #11 remains open for complete Permission CRUD, platform Resource.Action catalog/grant migration, full HTTP/rollback coverage and durable audit. Issue #9 remains open for bulk/raw write-path audit and remaining cross-tenant write acceptance.

- Post-merge [Backend main CI 39870192240](https://github.com/sholehamed/Shinera-backend/actions/runs/39870192240) also **succeeded** for merge commit `3c3e2fce`. The 13/13 SQL Server suite was verified on the final PR head, not automatically on main push.
