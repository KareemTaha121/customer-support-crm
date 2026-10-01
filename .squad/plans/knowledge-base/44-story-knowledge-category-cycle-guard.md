# Story 44 — Validate knowledge base category parents (Bug: BUG-08)

> Fix plan, implemented in `customer-support-crm-api` commit `e6bf4d7` (`develop`) and `customer-support-crm-web` commit `06c817a` (`main`). Paths and line numbers refer to those commits.
> Intake: [../../stories/knowledge-base/knowledge-category-cycle-guard/intake.md](../../stories/knowledge-base/knowledge-category-cycle-guard/intake.md)

## Prerequisites

- Story 29 — [29-story-knowledge-base-articles-and-search.md](29-story-knowledge-base-articles-and-search.md): `KnowledgeCategory`, `SaveKnowledgeCategoryHandler`.
- Story 43 — [../ticket-management/43-story-ticket-category-cycle-guard.md](../ticket-management/43-story-ticket-category-cycle-guard.md): `CategoryHierarchy` (`Common/Validation/CategoryHierarchy.cs`), `CATEGORY_CYCLE` and its en/ar messages.
- Frontend story 15 — [../frontend/15-story-knowledge-base-ui.md](../frontend/15-story-knowledge-base-ui.md): the staff category dialog.

---

## Story Goal

Before the fix, `POST /kb/categories` and `PUT /kb/categories/{id}` (`kb.manage`) had two problems:

- **Unknown `parentId`:** nothing checked it, so it reached the foreign key (`KnowledgeBaseConfiguration.cs` line 43, `Restrict`). `GlobalExceptionHandler` maps only unique violations to 409, so an FK violation (`23503`) became a **500**. The KB overview previously said 409; that was wrong.
- **Cycles:** `KnowledgeCategory.Update` rejected only `parentId == Id` (422 domain error). A category could be moved under one of its subcategories.

After the fix, a parent that is set (on create) or changed (on update) must:

1. **exist** → otherwise 404 `KB_CATEGORY_NOT_FOUND` "The parent category was not found.";
2. **not be the category or a descendant** → otherwise 400 `CATEGORY_CYCLE` on `parentId`. Self-parenting now also returns this 400, before the domain check runs.

**Deviation from the intake:** the intake suggested a field error for a missing parent. A 404 `KB_CATEGORY_NOT_FOUND` was built instead, to match ticket categories (404 `CATEGORY_NOT_FOUND` for a missing parent).

**Frontend change (`06c817a`):** the staff category dialog hid only the edited category from the parent picker. It now hides the whole subtree, as the ticket category dialog does. It also gained a `<mat-error>` on `parentId`. Without it, a matched field error marked the picker red with no text: `applyKbServerErrors` returns only unmatched messages for the toast.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Application/Common/Validation/CategoryHierarchy.cs` (from story 43): `CreatesCycle`, `CycleError`.
2. `src/CustomerSupportCrm.Application/Features/KnowledgeBase/KnowledgeBaseSlices.cs`: `SaveKnowledgeCategoryHandler` (127–163). The parent checks are at 144–158, before `category.Update` (160). `KnowledgeErrors.CategoryNotFound` is at line 26.
3. `src/CustomerSupportCrm.Domain/KnowledgeBase/KnowledgeArticle.cs`: `KnowledgeCategory` (226) and `Update` (265), which keeps its own `parentId == Id` guard.
4. `customer-support-crm-web/src/app/features/knowledge-base/staff/category-dialog.component.ts`: `<mat-error>` on `parentId` (57), `parents = excludeSubtree(...)` (88), `excludeSubtree` (141–157).
5. `customer-support-crm-web/src/app/features/knowledge-base/kb-server-errors.ts`: strips the `category.` prefix, then `applyServerErrors`.

---

## Backend Tasks

In `SaveKnowledgeCategoryHandler`, after the category is loaded or created and before `category.Update`:

```csharp
if (input.ParentId is { } parentId && parentId != category.ParentId)
{
    var parents = await db.KnowledgeCategories.AsNoTracking().ToDictionaryAsync(c => c.Id, c => c.ParentId, ct);
    if (!parents.ContainsKey(parentId))
    {
        throw new NotFoundException(KnowledgeErrors.CategoryNotFound, "The parent category was not found.");
    }

    if (CategoryHierarchy.CreatesCycle(category.Id, parentId, parents))
    {
        throw CategoryHierarchy.CycleError();
    }
}
```

On create, the new category is not in `parents`; it can still fail the existence check, but never the cycle check. On update to itself, the id is in `parents`, so the request returns `CATEGORY_CYCLE`.

Add `using CustomerSupportCrm.Application.Common.Validation;`.

**No changes to:** domain, contracts, migrations, resx (`KB_CATEGORY_NOT_FOUND` and `CATEGORY_CYCLE` already have en/ar), or docs.

## Frontend Tasks

In `category-dialog.component.ts`:

- Replace `parents = categories.filter(c => c.id !== self)` with `excludeSubtree(categories, self)`. That is a module-level function which grows an excluded set from the root through `parentId` links, the same algorithm as `ticket-category-dialog.component.ts` `parentOptions`.
- Add `<mat-error>{{ form.controls.parentId | formError }}</mat-error>` inside the parent `mat-form-field`.

---

## Test Plan

Test projects are **out of scope**. Neither repo has tests covering KB categories.

---

## Verification Steps

1. **Backend build:** the API was running locally and locking `bin/`, so the build ran as `dotnet build src/CustomerSupportCrm.Api/CustomerSupportCrm.Api.csproj -o <temp dir>`. At `e6bf4d7`: 0 warnings, 0 errors.
2. **Frontend build:** `cd customer-support-crm-web && npx ng build`. At `06c817a`: 0 errors, 0 warnings.
3. **Set up:** create KB categories A, then B with `parentId = A`, then C with `parentId = B`.
4. **Cycle:** `PUT /api/v1/kb/categories/A` with `parentId = C` → 400 `CATEGORY_CYCLE` on `parentId`. With `parentId = A` → the same 400.
5. **Unknown parent:** `POST /api/v1/kb/categories` with a random `parentId` → 404 `KB_CATEGORY_NOT_FOUND` (used to be a 500).
6. **Valid:** `PUT …/C` with `parentId = A` → 200. Renaming B without changing its parent → 200.
7. **UI:** in `/knowledge-base` → Categories, edit A. The parent picker doesn't list B or C. Not run while writing this plan: another session was running the API and dev server.

---

## Done Criteria

- [x] An unknown parent id is rejected (404, not 500).
- [x] Setting the category itself or a descendant as parent returns 400 `CATEGORY_CYCLE`.
- [x] The staff dialog hides the subtree and shows a `parentId` server error.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` and `ng build` pass with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to the next bug story.**
