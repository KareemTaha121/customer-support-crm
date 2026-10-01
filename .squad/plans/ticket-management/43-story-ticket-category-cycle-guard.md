# Story 43 — Prevent cycles in ticket category parents (Bug: BUG-07)

> Fix plan, implemented in `customer-support-crm-api` commit `cedce62` (`develop`). Paths and line numbers refer to that commit.
> Intake: [../../stories/ticket-management/ticket-category-cycle-guard/intake.md](../../stories/ticket-management/ticket-category-cycle-guard/intake.md)

## Prerequisites

- Story 26 — [26-story-ticket-conversation-and-categories.md](26-story-ticket-conversation-and-categories.md): `TicketCategory`, `SaveTicketCategoryHandler`.
- Story 42 — [../platform/42-story-inactive-organization-units.md](../platform/42-story-inactive-organization-units.md): the department check added to the same handler.
- All paths are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

`POST /ticket-categories` and `PUT /ticket-categories/{id}` (`tickets.categories_manage`) only checked that `parentId` exists. A category could be moved under itself or under one of its descendants (A → B → A), which breaks the tree for the category list, the pickers and reports.

After the fix, an update that **changes** the parent walks up the tree from the new parent. If it reaches the category itself, the request fails with **400 `CATEGORY_CYCLE`** on `parentId`. The category dialog (`customer-support-crm-web/src/app/features/tickets/ticket-category-dialog.component.ts`) already maps field errors onto the `parentId` control through `applyServerErrors`. Its parent picker already hides descendants (line 120), so the UI rarely sends one; the server now enforces the rule anyway.

**Deviation from the intake:** none.

Notes:

- A **new** category cannot create a cycle, because nothing points at its id yet, so only updates are checked.
- An unchanged parent is not re-checked.

The helper is shared with knowledge-base categories (BUG-08, plan 44).

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Application/Common/Validation/CategoryHierarchy.cs` (new): `CycleCode` (9), `CreatesCycle` (16–29), `CycleError` (32–33).
2. `src/CustomerSupportCrm.Application/Features/Tickets/TicketCategorySlices.cs`: `SaveTicketCategoryHandler`, with the cycle check at lines 70–77, just after the category is loaded on update.
3. `src/CustomerSupportCrm.Api/Middleware/GlobalExceptionHandler.cs`: `ToApiError` (70–77) localizes feature codes and camel-cases the field (`ParentId` → `parentId`).
4. `src/CustomerSupportCrm.Application/Resources/Messages.resx` / `Messages.ar.resx`: `CATEGORY_CYCLE`, next to `CATEGORY_IN_USE`.

---

## Backend Tasks

### 1 — Shared helper

Create `Common/Validation/CategoryHierarchy.cs` as an `internal static class`.

- `CycleCode = "CATEGORY_CYCLE"`.
- `CreatesCycle(Guid categoryId, Guid parentId, IReadOnlyDictionary<Guid, Guid?> parents)` follows `parents` up from `parentId`. It returns `true` when it reaches `categoryId`, which also covers `parentId == categoryId`. It also returns `true` when it revisits a node (a loop already present in the data), so a broken chain is never extended. It returns `false` at a root.
- `CycleError()` returns `ValidationException([ValidationFailure("ParentId", …) { ErrorCode = CycleCode }])`.

### 2 — Handler

In `SaveTicketCategoryHandler`, on **update**, after loading `category`:

```csharp
if (input.ParentId is { } newParent && newParent != category.ParentId)
{
    var parents = await db.TicketCategories.AsNoTracking().ToDictionaryAsync(c => c.Id, c => c.ParentId, ct);
    if (CategoryHierarchy.CreatesCycle(category.Id, newParent, parents))
    {
        throw CategoryHierarchy.CycleError();
    }
}
```

The category table is small, so one `(Id, ParentId)` read is enough. The existing "parent exists" check (404 `CATEGORY_NOT_FOUND`) still runs first. Add `using CustomerSupportCrm.Application.Common.Validation;`.

### 3 — Messages

- `Messages.resx`: "A category cannot be moved under itself or one of its subcategories."
- `Messages.ar.resx`: "لا يمكن نقل التصنيف تحت نفسه أو تحت أحد تصنيفاته الفرعية."

The English `CATEGORY_NOT_FOUND` entry is still missing; it is part of BUG-09.

**No changes to:** domain, contracts, migrations, docs or the frontend.

---

## Test Plan

Test projects are **out of scope**: `tests/` must not be modified. No existing test covers ticket categories.

---

## Verification Steps

1. **Build:** `dotnet build` at `cedce62` gives 0 warnings and 0 errors. While writing this plan the API was running locally and locking `bin/`, so the check used `dotnet build src/CustomerSupportCrm.Api/CustomerSupportCrm.Api.csproj -o <temp dir>`.
2. **Set up:** create categories A, then B with `parentId = A`, then C with `parentId = B`.
3. **Cycle:** `PUT /api/v1/ticket-categories/A` with `parentId = C` → 400, `errors[0] = { code: "CATEGORY_CYCLE", field: "parentId" }`. With `parentId = A` you get the same response.
4. **Valid move:** `PUT …/C` with `parentId = A` → 200.
5. **Unchanged parent:** `PUT …/B` keeping `parentId = A` while renaming → 200.
6. **Localization:** step 3 with `Accept-Language: ar` returns the Arabic message.

---

## Done Criteria

- [x] Setting the category itself or a descendant as parent returns 400 `CATEGORY_CYCLE` on `parentId`.
- [x] en/ar messages exist for `CATEGORY_CYCLE`.
- [x] The helper is reusable by knowledge-base categories (BUG-08).
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to the next bug story.**
