# System Admin — Frontend Stabilization and Real Backend Contract Audit

**Date:** 2026-10-24  
**Status:** Development branches / Draft PRs. **NOT MERGED**, not Production-approved.  
**Frontend:** [PR #4](https://github.com/sholehamed/Shinera-frontend/pull/4) — `fix/system-admin-real-api-e2e-20261024`  
**Backend:** [PR #32](https://github.com/sholehamed/Shinera-backend/pull/32) — `fix/system-admin-menu-category-integration-20261024`  
**Authoritative implementation-level page decision log:** [Frontend SYSTEM-ADMIN-PAGE-AUDIT](https://github.com/sholehamed/Shinera-frontend/blob/fix/system-admin-real-api-e2e-20261024/docs/SYSTEM-ADMIN-PAGE-AUDIT-2026-10-24.md)

## User-facing objective

Deliver the first actually navigable, authenticated System Admin interface connected to the real .NET Identity endpoints, without implying unfinished screens or unsafe System User writes are production-ready. The independent System context uses root-domain routes (/system), not tenant subdomains; the backend remains the sole source of authorization.

## Keep, repair, remove

| Surface | Decision | Why |
|---|---|---|
| System login + dashboard | Keep and repair | OpenIddict/CAPTCHA login, authenticated dashboard request, scoped Bearer interceptor, dedicated System-only route guard and 401 recovery |
| Tenant list/create/edit and overview | Keep and repair | Real System Tenant API; add API-backed tenant overview and remove obsolete back-navigation |
| System Users directory | Keep as **read-only** | UserCreate currently forces non-null static TenantId; UserUpdate unconditionally hashes Password again. Explicitly hide Create/Edit routes/actions until correct System user scope, type, password semantics and E2E exist |
| Role list + permission tree | Keep | Existing live backend endpoints; new real RoleCreate/RoleUpdate form replaces prior placeholder |
| Groups + group membership | Keep and repair | Proper paging body, explicit UserId field added in backend query, individually scoped audited member DELETE. No fictitious bulk-delete route |
| Modules, Resources, API/UiResources and Permission pages | Keep, integration follow-up | Existing nonempty UI/API implementations; UIResource API prefix corrected. Full browser CRUD tests not yet claimed |
| Menus and categories | Keep and repair | Dialog Create/Update state fixed; align backend PUT MenuCategories (ID in JSON), POST Menus/reorder. Backend update handler now targets MenuCategory, not User; category drag-reorder remains disabled pending API |
| Tenant detail Users/Roles/Groups/Permissions/Settings | **Remove now** | Five scaffold-only tabs had only `... works!` text, no real tenant-domain integration. Remove links/routes/components; implement real tenant administration as a separate planned slice |
| Tenant detail Overview | Keep and implement | Authenticated GET tenant by ID, real data and edit link |

## Backend contract corrections

Only three narrowly scoped changes in Draft Backend PR #32:
1. `MenuCategoryUpdateCommandHandler` now loads `context.MenuCategories` rather than `context.Users`.
2. `GroupMemberPagedListDto` explicitly exposes `UserId` (for audited `DELETE /system/Groups/{groupId}/members/{userId}`).
3. `SuperDashboard` now computes `TotalUsers`.

No new Production grants, signing configuration, tenant-scope bypass or migration flag change.

## Real full-stack validation — verified executable branch and documentation-only follow-up

| Gate | Evidence | Result |
|---|---|---|
| Angular development build / Vitest | [Frontend CI #40092435121](https://github.com/sholehamed/Shinera-frontend/actions/runs/40092435121) at frontend `aea2f27dd911c7daceaac4adb60198a621a6c86a` | **PASS, 15/15 unit tests** |
| Playwright browser + real .NET 10 API + ephemeral SQL Server 2022 | [Frontend E2E #40092435108](https://github.com/sholehamed/Shinera-frontend/actions/runs/40092435108) on the same frontend head | **PASS, 16/16 real full-stack tests** |
| Backend build & unit | [Backend CI #40091066192](https://github.com/sholehamed/Shinera-backend/actions/runs/40091066192) at backend `ca1d408ea02c5eefd5fcba7810f19928369cd6cd` | **PASS** |
| Backend SQL Server security | [Backend SQL CI #40091066164](https://github.com/sholehamed/Shinera-backend/actions/runs/40091066164) on same backend head | **PASS** |

The Playwright tests cover actual System admin browser login, protected dashboard and Tenant list, Role create/update/get/delete, and MenuCategory API create/update/get/delete. Tests are against disposable local E2E credentials/DB, **not mocked HTTP and not Production**. Legacy test coverage also includes unauthorized access/refresh-token behavior. Full coverage of System User write, GroupMember mutations and every legacy management page is not claimed.

**Latest recheck on frontend head `d5f08390356493449dd7675663b1274533b7188d` after frontend audit/E2E-doc changes:** [Build/Unit #40093675395](https://github.com/sholehamed/Shinera-frontend/actions/runs/40093675395) **PASS 15/15** and [actual .NET+SQL Server Playwright #40093675380](https://github.com/sholehamed/Shinera-frontend/actions/runs/40093675380) **PASS 16/16**. Counts above for the earlier `aea2f27d` head remain historical evidence; both sets passed. This central documentation change does not alter application code.

**Integration requirement before merge:** The full-stack workflow currently checks out the backend **Draft PR #32 branch** to exercise coordinated changes; after backend PR #32 merges, restore workflow checkout to `main`, run E2E again, and only then review/merge frontend PR #4. The latest CI verification predates documentation-only commits; these must not be represented as independently rerun test results without new CI.

## Remaining acceptance gates and recommended focus

- Correct backend **System User Create/Update** contract: no fixed TenantId, proper System user type, no unintended password reset, server validation, audit/authorization and actual API+browser tests. Keep UI write paths OFF until fixed.
- E2E the still-unverified CRUD screens (GroupMember read/mutation, Permission, Resources, modules, menu UI), and finalize startup/feature environmental configuration for eventual deployment.
- Revalidate menus and correct server-side effective menu role scoping for non-SuperAdmin accounts before delegating platform administration.
- No additional authentication/security infrastructure overhaul is needed for this first UI integration slice; keep Production granular enforcement OFF and P0 Issue #11 outstanding release gates separate.

**Release interpretation:** This documents a working, tested initial System Admin vertical slice **in Draft**, not a claim that all current admin UI screens are functionally complete or deployable to real customers.

## Same-stage extension — access-management resource screens (2026-10-24)

The original `Permission`, `ApiResource`, `UiResource` and `Resource` pages are **kept and repaired**, not re-created from scratch.

- System sidebar now adds a dedicated «مدیریت دسترسی» section and merges its links with persisted dynamic menu items without route duplicates. The server remains the source of authorization.
- Original scoped list endpoints `POST /system/{Permissions|ApiResources|UiResources}/{resourceId}/pagedList` remain. The backend Draft PR #32 adds corresponding root/global `POST /system/{Permissions|ApiResources|UiResources}/pagedList`, restricted by the existing `SystemAdministrator` policy; top-level System menu links no longer send `undefined` as a resource ID.
- CRUD delete actions are now wired, and forms follow actual enum and resource contracts (UI PAGE/ACTION/FIELD/WIDGET; API HTTP verb/source; Permission linked API/UI Resource array); ApiResource sync remains resource-scoped only.
- New cross-repo Playwright case creates disposable Module → Resource → ApiResource + UiResource → Permission, verifies global/scoped paged lists, live GET/PUT updates, permission resource associations and deletion. It also asserts the three System menu links are navigable. **Verified PASS:** [real full-stack E2E 40096193047](https://github.com/sholehamed/Shinera-frontend/actions/runs/40096193047) **17/17 Chromium tests** on frontend executable head `67acb0305586869e7d60b0966d18b77939a081e3`, including access-management menu, both global and scoped paged lists, and actual Permission/API/UI Resource SQL Server CRUD. [Angular Build/Unit 40096193049](https://github.com/sholehamed/Shinera-frontend/actions/runs/40096193049) **PASS 15/15**.
- [Frontend implementation audit](https://github.com/sholehamed/Shinera-frontend/blob/fix/system-admin-real-api-e2e-20261024/docs/SYSTEM-ADMIN-PAGE-AUDIT-2026-10-24.md). Both engineering PRs and Product PR #28 remain Draft. No production grants or rollout.

**Permission deletion order:** Real SQL Server E2E exposed intentional FK `409 Conflict` for a Permission still linked to ApiResource/UiResource. The valid cleanup workflow is audited Permission Update with `permissionResources: []` followed by Permission DELETE, then resource DELETE. Frontend reports blocked deletes instead of falsely showing success. The corrected unlink-first cleanup passed in [E2E 40096193047](https://github.com/sholehamed/Shinera-frontend/actions/runs/40096193047).
