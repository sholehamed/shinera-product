# Shinera Engineering Log — 2026-10-23

## Started: real SQL Server Identity security tests and ADR-003 review

- Created backend branch `security/sqlserver-authorization-regression-20261023` and [Draft Backend PR #7](https://github.com/sholehamed/Shinera-backend/pull/7).
- Added isolated SQL Server test project, GitHub Actions PR workflow and regression cases for revoked System admin role, inactive account, tenant filtering and tenant suspension.
- Started ADR-003 gap review. Current route-generated permission checker and the disabled middleware are not suitable for direct activation; backend must enforce stable Resource + Action + Scope keys, explicit Tenant/Branch/Own semantics, Platform separation and default deny.
- Created product documentation branch `docs/identity-authorization-sqlserver-20261023` and [detailed status record](IDENTITY-SQLSERVER-AUTHORIZATION-2026-10-23.md).
- [First SQL Server run 39854609746](https://github.com/sholehamed/Shinera-backend/actions/runs/39854609746) **passed 2/2 SQL Server integration tests** and applied real EF Core migrations on disposable databases; [Backend CI 39854609492](https://github.com/sholehamed/Shinera-backend/actions/runs/39854609492) also passed.
- Implemented explicit Tenant-scope Resource + Action authorization requirement and a default-deny DB-backed handler; supports direct/role/group grants while checking user and Tenant identity, active membership and resource/action key. No URL-derived permissions or assumptions about System accounts.
- Added third SQL Server regression covering valid/invalid resource action, revoked role grant, direct grant, cross-Tenant denial and tenant deactivation.
- **Still Draft; not merged.** Latest CI of permission handler and third test to be inspected and linked.

- **No code merged**; new SQL Server CI evidence to be attached after execution.
