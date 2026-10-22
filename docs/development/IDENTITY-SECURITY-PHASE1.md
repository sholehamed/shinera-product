# Shinera — Identity/System Security Hardening (Phase 1)

**Started:** 2026-10-22  
**State:** Scoped P0 baseline merged — complete Identity security hardening **still open**  
**Tracked risk:** [backend security issue #2](https://github.com/sholehamed/Shinera-backend/issues/2)

## Active branches and PRs

| Repo | Branch | PR | Goal |
|---|---|---|---|
| Backend | `security/identity-system-hardening-20261022` | [#3](https://github.com/sholehamed/Shinera-backend/pull/3) — **Merged** ([6d9432f8](https://github.com/sholehamed/Shinera-backend/commit/6d9432f8cd6645b2acb8fce864031f0020483fb8)) | Privileged System authorization baseline, CAPTCHA gate, development-only system bootstrap |
| Frontend | `test/identity-system-security-20261022` | [#2](https://github.com/sholehamed/Shinera-frontend/pull/2) — **Merged** ([af772b2](https://github.com/sholehamed/Shinera-frontend/commit/af772b2052f9c36a71099f006dffd50d6e224ced)) | Real API negative Playwright regression; backend checkout restored to `main` |
| Product | `docs/identity-security-phase1-20261022` | Document PR | Canonical work log, architecture constraints and verification evidence |

## Analysis and decisions

- Previous System management endpoints (Users, Roles, Tenants, Modules, Permissions, Menus, Groups, Resources, API/UI Resources, MenuCategories and Dashboard) mostly lacked an enforced authorization boundary. A few had only `.RequireAuthorization()`.
- Existing `PermissionMiddleware` was disabled and its checker required `tenant_id`, making it unsuitable for system users without a TenantId. Simply enabling it could bypass annotated endpoints or reject all platform users.
- **Current interim boundary:** protected management endpoint groups must require an authenticated System account (`account_type=system`) with no `tenant_id` and a `superadmin` role. A tenant user, including tenant Owner, is not a platform administrator.
- Anonymous `GET /api/System/Tenants/resolve` and public login/CAPTCHA remain public where explicitly intended.
- This coarse platform-level gate **does not replace** ADR-003 fine-grained Resource + Action + Scope permission assignments. That model must be completed separately, default-deny and database-backed.
- Login previously ignored the CAPTCHA validation result; password-flow login now rejects false before credential verification. Unsupported grant types and invalid/missing login context are explicitly rejected.
- Backend development seed initializes `superadmin` role membership for local seeded System admin and stops seeding a known-password administrator in Production. Existing **production** administrators require a separately reviewed role-mapping/bootstrap plan.

## Verification inventory

- Backend PR #3 and post-merge `main`: [CI run 39846613412](https://github.com/sholehamed/Shinera-backend/actions/runs/39846613412) — **19 xUnit passed; build successful**.
- Frontend PR #2 final [Full-stack CI run 39846647097](https://github.com/sholehamed/Shinera-frontend/actions/runs/39846647097): **9/9 Playwright tests passed** with Angular, real ASP.NET Core (backend `main`) and disposable SQL Server. Negative cases assert anonymous 401 across **all 12** protected System groups, tenant-mutation denial, malformed CAPTCHA `invalid_grant`, and missing login context. [Frontend CI run](https://github.com/sholehamed/Shinera-frontend/actions/runs/39846647016) covers build and 14 Vitest.
- Existing baseline: [10 xUnit](https://github.com/sholehamed/Shinera-backend/actions/runs/39841510200), [14 Vitest](https://github.com/sholehamed/Shinera-frontend/actions/runs/39841585176), [5 full-stack Playwright smoke](https://github.com/sholehamed/Shinera-frontend/actions/runs/39839196568).

## Remaining P0 work and limitations

1. **Next:** positive platform admin login and role permission tests with isolated test credentials; assert exact 401 vs 403 for anonymous vs authenticated non-admin. Negative E2E already passed.
2. Review active-token and refresh-token behavior after user deactivation, tenant suspension and role revocation. Existing refresh handler may reissue stale claims.
3. Review Logout/refresh revocation and OpenIddict userinfo consistency. Do not claim complete single-session security yet.
4. Dynamic API permission assignment enforcement (ADR-003) is **not** implemented by the coarse `SystemAdministrator` policy. Re-enable/refactor only after correct system/tenant-aware resolution, permission mapping and database tests.
5. Default development admin credentials still exist for local-only development; override `SHINERA_DEVELOPMENT_ADMIN_PASSWORD`, never expose development servers publicly. Rotate any legacy credentials, especially existing installations.
6. Replace development signing/encryption certificates with production-managed key material before deployment.
7. Add proper `WebApplicationFactory` integration tests and cover EF tenant query filters, branch scope and cross-tenant CRUD.

## Release and merge gates

- **Satisfied:** Backend PR #3 merged; frontend temporary backend branch ref was removed; final 9/9 Playwright tests ran against backend `main` and frontend PR #2 merged.
- **Not satisfied for production:** live System admin role/bootstrap migration; per-action ADR-003 authorization; refresh/token revocation checks; production certificate provisioning. Track via [Issue #4](https://github.com/sholehamed/Shinera-backend/issues/4) and [Issue #5](https://github.com/sholehamed/Shinera-backend/issues/5).
- A green CI run means only tested behaviors passed. It must not be described as full identity/permission security certification.
- Preserve reference to [ADR-001](../architecture/decisions/ADR-001-MULTI-TENANCY-STRATEGY.md), [ADR-002](../architecture/decisions/ADR-002-AUTHENTICATION-WITH-OPENIDDICT.md) and [ADR-003](../architecture/decisions/ADR-003-AUTHORIZATION-AND-PERMISSION-MODEL.md).


## 2026-10-22 — OpenIddict HTTPS requirement and E2E

The first negative API E2E run [39843886314](https://github.com/sholehamed/Shinera-frontend/actions/runs/39843886314) correctly verified anonymous management denial but found that the token endpoint rejected HTTP requests *at the OpenIddict transport layer* with `invalid_request` / "This server only accepts HTTPS requests". As a result, that run **did not execute** the intended CAPTCHA invalid-grant behavior.

Remediation: the backend now supports `Identity:AllowInsecureHttpForE2E` **only while `IHostEnvironment.IsDevelopment()`**. The frontend CI sets `Identity__AllowInsecureHttpForE2E=true` for disposable local Runner services. The default HTTPS requirement remains enforced for all other environments. **This is a test-only exception, not a recommended deployment setting.**

**Verified:** [final full-stack run 39846647097](https://github.com/sholehamed/Shinera-frontend/actions/runs/39846647097) asserted `invalid_grant` for malformed CAPTCHA against the merged backend `main` (using Development-only HTTP E2E opt-in).
