# Platform Permission CRUD — real HTTP matrix and expected failure mapping

**Date:** 2026-10-23  
**Tracking:** [Backend P0 Issue #11](https://github.com/sholehamed/Shinera-backend/issues/11)  
**Backend:** [PR #26](https://github.com/sholehamed/Shinera-backend/pull/26) · [merged commit `016186ae`](https://github.com/sholehamed/Shinera-backend/commit/016186ae3ff9f70e1179048164d0ea2f143bf6f2)  
**Status:** Backend PR #26 reviewed and squash-merged; latest PR-head CI passed

## Problem seen in the current code

The existing System Permission endpoints enforce live SystemAdministrator access and support optional Resource.Action permissions, but several expected CRUD errors did not have explicit HTTP contracts. A missing Permission read threw a generic `Exception("not found")`; create validation was empty; and the CQRS `IDispatcher` used by these endpoints does not automatically execute the nested FluentValidation validators. Missing Resource/linked-resource IDs could thus emerge as database exceptions rather than an intentional client error.

## Scoped fix

The Permission write executor checks current persisted administrator authorization within the existing fixed Master write gateway before inspecting request state. It now distinguishes malformed code/name/resource/link input (400), missing target IDs (404), code conflicts and recognized SQL FK/unique conflicts (409), while preserving reserved deployment-managed permission catalog 403. Unexpected exceptions remain visible as failures rather than silently rewritten.

It does not enable Production Resource.Action enforcement, migrate any Master grants, broaden tenant visibility or add API bypasses. A Development + E2E flag is required for the isolated Resource fixture; normal deployments do not receive it.

## Live OpenIddict HTTP acceptance matrix

| Operation | Expected HTTP |
| --- | --- |
| Anonymous Permission GET/POST | 401 |
| Ordinary System or Tenant caller | 403 |
| Reserved ROLES/PERMISSIONS catalog create/edit/delete/state | 403 |
| Valid scoped Create / persisted GET / Update / ChangeState / Delete | 201 / 200 / 204 / 204 / 204 |
| GET/PUT/DELETE/PATCH unknown Permission | 404 |
| Missing/unknown Resource, empty Code or Name, invalid API/UI links | 400 |
| Duplicate Permission Code (normalized) | 409 |
| GET deleted Permission | 404 |

The tests make actual signed OpenIddict HTTP requests to the .NET API with a disposable SQL Server. Permission state is read back through the public endpoint after successful writes.

## Review limitation: concurrent duplicates

The verified HTTP 409 duplicate-code behavior is **sequential**. The current Permission table does not have an enforced unique index on normalized Code. Parallel create/update requests could pass the pre-write check and create duplicate codes until a reviewed data cleanup/index migration and concurrency regression are added. This is tracked under the separate concurrency/Production rollout acceptance work for Issue #11; this slice does not claim database-level uniqueness or production-ready concurrency guarantees.

## Remaining acceptance work

This slice focuses on Permission CRUD error contracts and does not close Issue #11. Remaining: durable denied-attempt audit, reviewed concurrency/deadlock and transactional fault injection, safe Production Master grant inventory/bootstrap/backfill/cutover/rollback, and final ADR-003 acceptance review.

**Verified latest backend PR-head evidence:** backend commit `b443883b3f19d5ba6b6b8eed87d7601ad384404c` passed [Build/Unit run 40014800301](https://github.com/sholehamed/Shinera-backend/actions/runs/40014800301) and [SQL Server + real OpenIddict HTTP run 40014800305](https://github.com/sholehamed/Shinera-backend/actions/runs/40014800305). Logs confirm **20/20 SQL Server regression tests** and **89/89 signed HTTP status checks**, including persisted Permission CRUD and previous stale-JWT/revoked refresh regression. Initial SQL Server CI on the first implementation head failed one legacy direct-gateway test because HTTP shape validation was incorrectly required for non-HTTP callers; this was corrected on the latest head without weakening reserved catalog or live actor checks. **Final security review was completed and Backend PR #26 squash-merged** at [`016186ae`](https://github.com/sholehamed/Shinera-backend/commit/016186ae3ff9f70e1179048164d0ea2f143bf6f2). These checks are PR-head evidence, **not independent post-merge SQL Server E2E, Production or comprehensive global HTTP error-contract verification**.
