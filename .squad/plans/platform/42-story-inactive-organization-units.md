# Story 42 — Reject assignment to inactive branches and departments (Bug: BUG-06)

> Fix plan, implemented in `customer-support-crm-api` commit `487e078` (`develop`). Paths and line numbers refer to that commit.
> Intake: [../../stories/platform/inactive-organization-units/intake.md](../../stories/platform/inactive-organization-units/intake.md)

## Prerequisites

- Story 21 — [21-story-organization-context-and-notifications.md](21-story-organization-context-and-notifications.md): branches, departments, user scopes, `OrganizationErrors`, `EnsureCanAssign`.
- Stories 23, 25, 26 and 28: the customer, ticket, category, SLA policy and assignment rule writes.
- All paths are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

`OrganizationErrors.InactiveUnit = "ORGANIZATION_UNIT_INACTIVE"` was defined but never thrown.

**State before the fix.** The intake overstated the problem, so here is what the code actually did:

| Write | Before | After |
|---|---|---|
| Customer create; customer move (`PUT /customers/{id}` with a new unit) | Inactive unit → **404** `BRANCH_NOT_FOUND` / `DEPARTMENT_NOT_FOUND` (`OrganizationUnits` in `CustomerProfileSlices.cs`) | 422 `ORGANIZATION_UNIT_INACTIVE`; a missing unit stays 404 |
| Ticket create, including the category's default department; `POST /tickets/{id}/transfer` | Same 404 | 422 |
| User scopes (`PUT /users/{id}/scopes`, create user with scopes) | **No active check** | 422 for any new scope in an inactive unit |
| Ticket category `defaultDepartmentId` | **No check** (not even existence) | 404 / 422 when set or changed |
| SLA policy `departmentId` | **No check** | 404 / 422 when set or changed |
| Assignment rule `setDepartmentId` | **No check** | 404 / 422 when set or changed |
| New department in a deactivated branch | Allowed | 422; editing an existing department is still allowed |

Rule for all of them: **only units that change are checked**. A record already in a deactivated unit can still be read and updated. User scopes keep the scopes the user already has.

**Deviations from the intake:**

- The 404 for a *missing* branch or department is kept, with the existing codes. The new 422 applies only to units that exist but are inactive.
- The intake said "customers, tickets, user scopes and categories". SLA policy departments, assignment rule targets and new departments were added, because they are the other writes that place data in a unit.
- Not changed:
  - **Match filters on rules** (`MatchDepartmentId`): they don't place data in a unit.
  - **External customer upsert by `branchCode`** (`IntegrationSlices.cs` ≈340): it still looks up only active branches and returns 404.
  - **Inbound routing** (`DefaultUnitAsync`): it already picks active units only.
- **Assignment engine:** a rule whose target department is later deactivated is already safe. `AssignmentEngine` only moves a ticket when the department is active (`Features/Sla/SlaEngine.cs` line 67).

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Application/Common/Authorization/AccessScope.cs`: `OrganizationUnits` (126–164), with `EnsureValidAsync`, `EnsureDepartmentActiveAsync` (148) and `Inactive()` (162). `OrganizationErrors` follows it.
2. `src/CustomerSupportCrm.Application/Features/Customers/CustomerProfileSlices.cs` lines 67 and 143. These are callers; the old private `OrganizationUnits` class was removed from this file.
3. `src/CustomerSupportCrm.Application/Features/Tickets/TicketCommandSlices.cs` lines 141 (create) and 247 (transfer).
4. `src/CustomerSupportCrm.Application/Features/Users/Common/UserQueries.cs`: `ResolveScopesAsync` (91–124), now with the optional `current` parameter.
5. `src/CustomerSupportCrm.Application/Features/Users/Administration/UserAdministration.cs`: `SetUserScopesHandler` passes the user's current scopes.
6. `src/CustomerSupportCrm.Application/Features/Tickets/TicketCategorySlices.cs` lines 69–81.
7. `src/CustomerSupportCrm.Application/Features/Sla/SlaAdministration.cs` lines 98–101 (policy) and 213–217 (assignment rule).
8. `src/CustomerSupportCrm.Application/Features/Departments/DepartmentEndpoints.cs` lines 36–43.
9. `src/CustomerSupportCrm.Application/Common/Exceptions/ServiceUnavailableException.cs` line 10: `UnprocessableException` → 422 (`GlobalExceptionHandler.StatusFor`).
10. `src/CustomerSupportCrm.Application/Resources/Messages.resx` / `Messages.ar.resx`.

---

## Backend Tasks

### 1 — Shared helper

Move `OrganizationUnits` from `CustomerProfileSlices.cs` into `AccessScope.cs` as an `internal static class`.

`EnsureValidAsync(db, branchId, departmentId, ct)`:

- Branch missing → `NotFoundException(BRANCH_NOT_FOUND)`.
- Department missing, or not in that branch → `NotFoundException(DEPARTMENT_NOT_FOUND)`.
- Either one inactive → `Inactive()`.

`EnsureDepartmentActiveAsync(db, departmentId, ct)`:

- One query returns the department's `IsActive` and whether its branch is active.
- Missing → 404 `DEPARTMENT_NOT_FOUND`.
- Either one inactive → `Inactive()`.

`Inactive()` returns `new UnprocessableException(ORGANIZATION_UNIT_INACTIVE, "The branch or department is inactive.")`.

Customer and ticket callers keep calling `OrganizationUnits.EnsureValidAsync`. The namespace is resolved through the `Common.Authorization` using they already have.

### 2 — User scopes

`ResolveScopesAsync(db, scopes, ct, current = null)` keeps the existing existence and branch-membership check (400 `DEPARTMENT_NOT_IN_BRANCH` on `Scopes`). It then rejects any requested scope **not in `current`** whose branch or department is inactive, with `OrganizationUnits.Inactive()`.

`SetUserScopesHandler` passes the user's current `(BranchId, DepartmentId)` pairs. `CreateUserHandler` passes none, so every scope must be active.

### 3 — Department-only references

Call `EnsureDepartmentActiveAsync` in these places:

- **Ticket category:** on create when `defaultDepartmentId` is set, and on update when it changes.
- **SLA policy:** on create when `departmentId` is set, and on update when it changes. Compare against `policy.DepartmentId` **before** `policy.Update`.
- **Assignment rule:** the same for `setDepartmentId`, compared against `rule.SetDepartmentId` before `rule.Update`.

Clearing a reference (null) is never checked. Add `using CustomerSupportCrm.Application.Common.Authorization;` to `TicketCategorySlices.cs` and `SlaAdministration.cs`.

### 4 — Departments

`SaveDepartmentHandler` reads the branch's `IsActive`; a missing branch is still 404. On **create** in an inactive branch, it throws `Inactive()`.

### 5 — Messages

- `Messages.resx` (after `OUT_OF_SCOPE`): add `ORGANIZATION_UNIT_INACTIVE` "The branch or department is inactive.", `BRANCH_NOT_FOUND` "The branch was not found." and `DEPARTMENT_NOT_FOUND` "The department was not found." Before this, English had none of them.
- `Messages.ar.resx`: `BRANCH_NOT_FOUND` becomes "الفرع غير موجود." and `DEPARTMENT_NOT_FOUND` becomes "القسم غير موجود.". Both used to add "or inactive"; that is now a separate code. `ORGANIZATION_UNIT_INACTIVE` was already present.

**No changes to:** contracts, domain, migrations or the frontend. The web app shows the server's localized message for unknown codes.

---

## Test Plan

Test projects are **out of scope**: `tests/` must not be modified. No existing test covers organization units.

---

## Verification Steps

1. **Build:** in `customer-support-crm-api/`, run `dotnet build`. At `487e078`: 0 warnings, 0 errors.
2. **Set up:** create branch B2 with department D2, then deactivate D2 (`POST /departments/{id}/deactivate`).
3. **Customer:** `POST /customers` with `branchId=B2, departmentId=D2` → 422 `ORGANIZATION_UNIT_INACTIVE`. A random department id → 404 `DEPARTMENT_NOT_FOUND`.
4. **Ticket:**
   - `POST /tickets/{id}/transfer` to D2 → 422.
   - A customer already in D2: `PUT /customers/{id}` without changing the unit → 200.
5. **User scopes:**
   - `PUT /users/{id}/scopes` adding `{branchId: B2, departmentId: D2}` → 422.
   - A user who already had that scope, with another scope added → 200, and the old scope is kept.
6. **Category / SLA / rule:**
   - Setting `defaultDepartmentId`, `departmentId` or `setDepartmentId` to D2 → 422.
   - Saving an existing one that already points at D2 → 200.
7. **Department:** deactivate B2 (`POST /branches/{id}/deactivate`).
   - `POST /departments` in B2 → 422.
   - `PUT` on an existing department in B2 → 200.
8. **Localization:** the 422 with `Accept-Language: ar` → "الفرع أو القسم غير نشط.", and in English "The branch or department is inactive."

---

## Done Criteria

- [x] Assigning an inactive branch or department returns 422 `ORGANIZATION_UNIT_INACTIVE` (customers, tickets, user scopes, categories, SLA policies, assignment rules, new departments).
- [x] Existing records in inactive units can still be read and updated without changing the unit.
- [x] en/ar messages exist for the code (and for `BRANCH_NOT_FOUND` / `DEPARTMENT_NOT_FOUND`).
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to the next bug story.**
