# Permission uniqueness, concurrency, rollback and deadlock — P0 Issue #11

**Date:** 2026-10-24  
**Backend:** [Draft PR #28](https://github.com/sholehamed/Shinera-backend/pull/28)  
**Issue:** [Backend P0 #11](https://github.com/sholehamed/Shinera-backend/issues/11)  
**Status:** Final-head CI passed. Draft / final security and migration review pending. Not merged.

## Current schema risk

The existing `Permissions.Code` column is `nvarchar(max)` and **does not have a normalized database unique constraint**. HTTP Permission CRUD currently checks normalized code duplicates before a write; two concurrent requests can both pass that check. HTTP 409 on sequential duplicate input is not a durable concurrency guarantee.

## Proposed invariant and rollout

A new SQL Server migration adds an **unmapped persisted computed column** `NormalizedCode`, using `UPPER(LTRIM(RTRIM(Code))) COLLATE Latin1_General_100_CI_AS` (converted to `nvarchar(100)`), and `UX_Permissions_NormalizedCode` as a **global unique index**, including inactive Permission rows. A database CHECK constraint enforces trimmed Code length from 1 through 100. This does not rely on a race-prone application-level `AnyAsync` alone.

**Important Production startup gate:** The backend previously ran `Database.MigrateAsync()` automatically in `DbSeeder.SeedAsync`. The PR now defers this new uniqueness migration in `Production` by default, continuing only through the last approved `20261023210000_platformaccessdenialaudit` migration. Its schema change requires explicit `Identity:PlatformPermissions:AllowCodeUniquenessMigration=true` under a supervised rollout after preflight, backup and controlled writes. The flag may be turned off again after successful rollout; the startup gate does **not** downgrade an already-applied index. Development/disposable CI still applies all migrations, and future migrations after the gated one are also deferred on Production until the gate is approved. Tests cover dirty legacy data with flag OFF, fail-closed opt-in with flag ON, manual cleanup, and post-rollout no-downgrade behavior.

The migration first checks for blank, NULL, overlong or duplicate normalized codes and **aborts safely rather than deleting or merging permissions**. Permission IDs are referenced by role and direct user grants. Existing duplicate remediation therefore requires explicit analysis and migration of references; no automatic winner is selected. The SQL preflight query in backend `scripts/sql/platform-permission-code-preflight.sql` is read-only and must be run before any production deployment. Prevent writes between clean inventory and migration, and prepare a database backup, verified rollback and maintenance window. Do not treat the migration's existence as authorization to deploy.

This index remains globally scoped because Permission definitions are shared references, whereas fixed Master Tenant is a **role/grant definition storage scope**, not the TenantId of System users. Production granular permissions remain OFF.

## Real SQL Server regression matrix

| Scenario | Expected outcome |
| --- | --- |
| Simultaneous create with identical code modulo case/outer whitespace | Exactly one commits; other receives SQL duplicate violation 2601/2627 |
| Insert with same normalized code when previous permission is inactive | Still rejected |
| Insert then transaction rollback | Row disappears; a later insert of the same code succeeds |
| Opposite-order updates on two rows under two connections and transactions | SQL Server selects one deadlock victim (1205); victim's earlier updates roll back; winner commits |
| Legacy normalized duplicates before index migration | Migration refuses to proceed; no Permission/grant ID auto-rewritten |
| Signed OpenIddict live HTTP Permission CRUD | Existing 89-check matrix remains green; duplicate conflict maps to 409 |

All tests used disposable SQL Server and actual EF migrations, not mocks or EF InMemory. **Final backend head `bd1ff8a0c3cadf0870f2383802bbf6ffce6a5875`** passed [Build/Unit CI 40023139711](https://github.com/sholehamed/Shinera-backend/actions/runs/40023139711) and [SQL Server + signed OpenIddict HTTP CI 40023139664](https://github.com/sholehamed/Shinera-backend/actions/runs/40023139664): **24/24 SQL Server tests and 89/89 real HTTP checks**, including the prior stale-JWT, persistence and denied-access audit assertions.

## Unfinished P0 acceptance gates

- [x] Exact final-branch Build/Unit, SQL migration, isolation and signed HTTP tests passing (CI links above).
- Review migration idempotency, backup/recovery, normalized collation and legacy-data preflight failures.
- [ ] Deadlock determinism and concurrency test stability across repeated independent CI runs; a single CI pass is not an extended stress guarantee.
- More extensive transaction fault injection for audited grant mutations, outbox/audit rollback, SQL outage, and realistic load/throughput.
- Production Master grant inventory/bootstrap/backfill/cutover/rollback and final ADR-003 acceptance.

**Issue #11 remains OPEN** and the backend and product PRs should stay Draft until final review/merge.
