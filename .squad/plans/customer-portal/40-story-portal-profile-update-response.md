# Story 40 — PUT /portal/me returns the updated name (Bug: BUG-04)

> Fix plan, implemented in `customer-support-crm-api` commit `1ad5302` (`develop`). Paths and line numbers refer to that commit.
> Intake: [../../stories/customer-portal/portal-profile-update-response/intake.md](../../stories/customer-portal/portal-profile-update-response/intake.md)

## Prerequisites

- Story 30 — [30-story-portal-accounts-and-authentication.md](30-story-portal-accounts-and-authentication.md): `CustomerAccount`, `PortalSessions.ProfileAsync`, the `/portal/me` endpoints.
- All paths are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

Before the fix, `PUT /portal/me` updated `Customer.Name`, while `PortalSessions.ProfileAsync` reads the name from `CustomerAccount.DisplayName`, so the portal kept showing the old name:

- in the response;
- in the next `GET /portal/me`;
- in the name inside new customer tokens (`IssueAsync`).

After the fix, the endpoint renames the account in the same `SaveChanges` as the customer record. The response, the next profile read and the next sign-in all show the new name.

**Deviation from the intake:** none.

**Not in scope, found while fixing:**

- **Staff rename has the same drift.** `PUT /customers/{id}` (`Features/Customers/CustomerProfileSlices.cs` line 165) updates `Customer.Name` but not the portal account, so after a staff rename the portal still shows the old name. Fixing it means calling `CustomerAccount.Rename` there too, or making the profile read `Customer.Name`. Listed in the overview's Known gaps.
- **An existing token keeps the old name claim** until the customer signs in again. That is acceptable, because tokens last at most 8 hours.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Domain/Customers/CustomerAccount.cs`:
   - `Rename` (154), the shared `ValidName` (169–178), `New` (156–167).
   - The name rule: trimmed, non-empty, ≤ `Customer.NameMaxLength`; otherwise `DomainException(INVALID_ACCOUNT)`.
2. `src/CustomerSupportCrm.Application/Features/CustomerPortal/PortalAccountSlices.cs`:
   - `PortalSessions.ProfileAsync` (39–50), which reads `a.DisplayName`.
   - `PUT /me` (277–294).
3. `customer-support-crm-web/src/app/features/customer-portal/auth/portal-profile.page.ts` line 118. The page already stores the PUT response (`portal.updateProfile(profile)`), so it needs no frontend change.

---

## Backend Tasks

### 1 — Domain

In `CustomerAccount`:

- Move the name validation from `New` into `private static string ValidName(string displayName)`, so the rule and the `INVALID_ACCOUNT` error are unchanged.
- Add:

  ```csharp
  public void Rename(string displayName) => DisplayName = ValidName(displayName);
  ```

- `New` now sets `DisplayName = ValidName(displayName)`.

### 2 — Endpoint

In `PUT /portal/me`, after `record.UpdateProfile(...)`, load the tracked account (`db.CustomerAccounts.SingleAsync(a => a.Id == customer.AccountId)`), call `account.Rename(request.Name)`, then the existing single `SaveChangesAsync`. The response is unchanged (`ProfileAsync`).

The endpoint's inline check (non-blank, ≤ `Customer.NameMaxLength`, language `en`/`ar`) runs first. So `Rename` cannot fail there, apart from a name that is only whitespace, which both checks already reject.

**No changes to:** contracts, migrations, resx, docs or the frontend.

---

## Test Plan

Test projects are **out of scope**: `tests/` must not be modified. No existing test covers the portal.

---

## Verification Steps

1. **Build:** in `customer-support-crm-api/` run `dotnet build`. At `1ad5302`: 0 warnings, 0 errors.
2. **Update:** sign in to the portal (`POST /api/v1/portal/auth/login`) and export `CTOKEN`. Then call `curl -X PUT …/api/v1/portal/me -H "Authorization: Bearer $CTOKEN" -H "Content-Type: application/json" -d '{"name":"New Name","language":"ar"}'`. The response has `displayName: "New Name"`.
3. **Read back:** `GET /portal/me` shows `New Name`. `GET /customers/{id}` as staff shows the same name.
4. **Sign-in:** sign in again. The session's profile shows `New Name`.
5. **Invalid:** `{"name":"   ","language":"en"}` → 400, as before.

---

## Done Criteria

- [x] The `PUT /portal/me` response and a following `GET /portal/me` show the new name.
- [x] Account creation and rename share one name rule.
- [ ] Staff rename (`PUT /customers/{id}`) also updates the portal account. Not in this story; see Known gaps.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to the next bug story.**
