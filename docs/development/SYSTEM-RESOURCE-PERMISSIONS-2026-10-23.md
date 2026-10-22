# Shinera — ADR-003 Platform Permissions, Branch/Own Roadmap

**Date:** 2026-10-23  
**Status:** In progress — draft PRs, no merge  
**Architecture:** [ADR-003](../architecture/decisions/ADR-003-AUTHORIZATION-AND-PERMISSION-MODEL.md), [ADR-001](../architecture/decisions/ADR-001-MULTI-TENANCY-STRATEGY.md)  
**Code:** [Backend PR #8](https://github.com/sholehamed/Shinera-backend/pull/8)  
**Tracking:** [Backend P0 Issue #5](https://github.com/sholehamed/Shinera-backend/issues/5)

## Current baseline

- [Backend PR #7](https://github.com/sholehamed/Shinera-backend/pull/7) merged real SQL Server authorization tests and opt-in **Tenant-only** `RequireTenantPermission(resource, action)` with database grants.
- Existing privileged System API routes are secured by `SystemAdministrator` and live role/account checks; no fine-grained System `Resource.Action` permission requirements have yet been attached to their endpoints.
- In the current persistence schema, System **users** have `TenantId == null`. However platform Role, RolePermission and UserPermission *definitions/assignments* use a fixed legacy Master Tenant id (`DefaultSeedData.MasterTenant.Id`) for storage. This is a transitional persistence detail, **not** an architectural claim that platform users belong to a parent Tenant.
- The current backend contains no completed BranchMembership/Staff/Appointment domain entities or owning-resource resolvers needed to implement ADR-003 Branch/Own. Do not simulate broad Branch/Own access from role name, `CreatedBy` or a route parameter.

## Implemented in Backend PR #8

1. `SystemPermissionRequirement(resource, action)` and `RequireSystemPermission(resource, action)` provide explicit, validated stable permission metadata on Minimal API endpoints, reusing the Tenant permission key normalization.
2. `SystemPermissionHandler` defaults to deny. It requires a valid authenticated `account_type=system` claim, **no** tenant_id claim, valid subject id, live active System user with `User.TenantId == null`, and active trusted platform definition scope.
3. Assignments are read only from `DefaultSeedData.MasterTenant.Id` and must have matching TenantIds through direct permission, UserRole→RolePermission, or UserGroup→GroupRole→RolePermission paths. Arbitrary tenant role/permission grants cannot become platform permissions.
4. A platform `superadmin` role name alone does **not** automatically imply all Resource.Action grants. Access requires a matching active database permission assignment. Explicit user denial (`IsGranted=false`) wins, respecting the **existing schema behavior**. ADR-003 defers *introducing* explicit deny as a new generalized concept; the project already has this legacy flag, so this slice preserves its behavior pending a dedicated ADR/schema reconciliation.
5. Two new SQL Server integration tests verify direct and inherited platform grants, default deny and role revocation, cross-tenant context denial, inactive System account/master scope and GroupRole tenant consistency.

**Safety:** This handler is registered but not automatically applied to existing privileged System groups, so administrators retain their current `SystemAdministrator` protection until the permission catalog, bootstrap/migration and real HTTP 401/403 tests are ready. The older URL-derived PermissionMiddleware remains disabled.

## Follow-up implementation order

1. Define stable platform Resource/Action catalog (initially System Tenants, Users, Roles, Permissions) and a controlled upgrade path assigning grants to existing platform roles in the fixed storage scope. Review bootstrap privileges separately.
2. Add `RequireSystemPermission` to a deliberately selected endpoint together with the existing base System gate, test 401/403/authorized calls against real ASP.NET Core and SQL Server, then roll out progressively across privileged groups.
3. When Branch & BranchMembership persistence is implemented, create explicit `BranchScopeRequirement` that confirms Branch belongs to active Tenant, User has current membership/access and PermissionAssignment.scope==Branch; default deny when any evidence is missing.
4. When ownership semantics exist for concrete business resources, implement resource-specific `OwnScopeRequirement` and resolvers (e.g. Appointment.Staff.UserId) rather than `CreatedBy` or role guesses.
5. Keep `Child` reserved until concrete hierarchy requirements exist. Subscription entitlement remains separate from authorization.
6. Build SQL Server/HTTP integration tests for cross-Tenant/Branch impersonation, direct grant/revocation, disabled user/role, scope changes, token replay and fine-grained System APIs.

## Verification ledger

| Gate | Evidence | Current result |
|---|---|---|
| Backend CI (Release Build + xUnit) | [Final head run 39859081756](https://github.com/sholehamed/Shinera-backend/actions/runs/39859081756) | **21/21 passed**, Release Build successful |
| Real SQL Server integration | [Final head run 39859082403](https://github.com/sholehamed/Shinera-backend/actions/runs/39859082403) | **6/6 passed** against disposable SQL Server with EF Core migrations |
| Full-stack authorized System API endpoints under new requirement | Not yet activated | Not implemented |
| Branch and Own authorization | Current domain persistence incomplete | Design/dependency only |

Do not merge or describe complete until the latest GitHub Actions run reports success, security review confirms tenant-scope containment and the scope boundaries above remain explicit.

## Review findings and safety debt

- Cross-Tenant escalation regression expanded: a System user is explicitly given a foreign Tenant role and valid permission assignment; this must **not** confer platform permission despite carrying the same Resource.Action key.
- [Security P0 issue #9](https://github.com/sholehamed/Shinera-backend/issues/9): `TenantSaveChangesInterceptor` currently has write-scope and TenantId immutability checks commented out. **This is outside the new read-side authorization handler.** It must be fixed with an explicit, audited platform operation path and real SQL Server tests before claiming safe tenant writes.
- A stored user-level `IsGranted=false` overrides inherited System grants because that is the current database behavior. ADR-003 defers introducing generalized explicit Deny as a **new** model; no migration/ADR change was made in this slice.
- The existing SystemAdministrator policy remains applied to all current System management endpoint groups; the new `SystemPermissionRequirement` is opt-in and not yet activated on those groups.
