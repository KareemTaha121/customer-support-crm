# Story 46 — Customer renames by staff or integrations reach the portal account name (Bug: BUG-10)

> Fix plan, implemented in `customer-support-crm-api` commit `dca992c` (`develop`). Paths and line numbers refer to that commit.
> Intake: [../../stories/customer-management/staff-rename-portal-account-sync/intake.md](../../stories/customer-management/staff-rename-portal-account-sync/intake.md)

## Prerequisites

- Story 23 — [23-story-customer-profiles-and-contacts.md](23-story-customer-profiles-and-contacts.md): `UpdateCustomerHandler`.
- Story 40 — [../customer-portal/40-story-portal-profile-update-response.md](../customer-portal/40-story-portal-profile-update-response.md): `CustomerAccount.Rename`, plus the fix for `PUT /portal/me`. This story was found while fixing that one.
- Story 36 — [../integrations/36-story-api-keys-webhooks-and-providers.md](../integrations/36-story-api-keys-webhooks-and-providers.md): the external customer upsert.
- All paths are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

The portal profile (`PortalSessions.ProfileAsync`) and new customer tokens (`IssueAsync`) both read `CustomerAccount.DisplayName`. Three writes rename a customer:

| Write | Before | After |
|---|---|---|
| `PUT /portal/me` | Fixed in BUG-04 (renames the caller's account) | Unchanged |
| `PUT /customers/{id}` (staff, `customers.update`) | Portal accounts kept the old name | In-step accounts follow |
| `PUT /external/customers/{system}/{externalId}` (API key, existing customer) | Same | In-step accounts follow |

**Why not every account.** A customer can have several portal accounts, because the email is unique per account, not per customer. Self-registration links to an existing customer by email but keeps the registrant's own name (`CustomerAccount.Register(…, input.Name, …)`), while a staff grant uses `customer.Name`.

So only accounts whose `DisplayName` equals the customer's **previous** name are renamed: those are the ones that were in step with it. A contact who registered under their own name for a company customer keeps that name.

**Deviation from the intake:** none.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Application/Features/CustomerPortal/PortalAccountNames.cs` (new): `FollowCustomerRenameAsync` (16–30).
2. `src/CustomerSupportCrm.Domain/Customers/CustomerAccount.cs`: `Rename` (from BUG-04), which shares the name rule with account creation.
3. `src/CustomerSupportCrm.Application/Features/Customers/CustomerProfileSlices.cs`: `UpdateCustomerHandler`. The call is at line 151, after `UpdateProfile` / `SetStatus`; `before.Name` is captured just above it.
4. `src/CustomerSupportCrm.Application/Features/Integrations/IntegrationSlices.cs` lines 354–359: the external upsert's update branch.
5. `src/CustomerSupportCrm.Application/Features/CustomerPortal/PortalAccountSlices.cs`: `Register` (line 116) and `CreateByStaff` (352), which show how account names are set.

---

## Backend Tasks

### 1 — Helper

Create `Features/CustomerPortal/PortalAccountNames.cs` as an `internal static class`:

```csharp
public static async Task FollowCustomerRenameAsync(IApplicationDbContext db, Guid customerId, string previousName, string newName, CancellationToken ct)
```

- Returns at once when `previousName == newName`, so an unchanged name never touches accounts.
- Loads tracked accounts where `CustomerId == customerId && DisplayName == previousName` and calls `account.Rename(newName)` on each.
- The caller's existing `SaveChangesAsync` persists the changes. Both names come from `Customer.Name`, which the domain has already trimmed and validated against the same `Customer.NameMaxLength` that `Rename` uses.

### 2 — Callers

- **`UpdateCustomerHandler`:** after `UpdateProfile`, call `FollowCustomerRenameAsync(db, customer.Id, before.Name, customer.Name, ct)`. Add `using CustomerSupportCrm.Application.Features.CustomerPortal;`.
- **External upsert:** in the update branch, capture `previousName = customer.Name` before `UpdateProfile`, then call `CustomerPortal.PortalAccountNames.FollowCustomerRenameAsync(...)`. That is the same namespace-relative style the file already uses for `Common.Authorization.OrganizationErrors`.

**No changes to:** contracts, domain, migrations, resx, docs or the frontend.

---

## Test Plan

Test projects are **out of scope**: `tests/` must not be modified.

---

## Verification Steps

1. **Build:** `dotnet build src/CustomerSupportCrm.Api/CustomerSupportCrm.Api.csproj -o <temp dir>` at `dca992c` gives 0 warnings and 0 errors. The API was running locally and locking `bin/`.
2. **Staff rename:** grant portal access to customer "Acme" (`POST /customers/{id}/portal-access`); the account is named "Acme". Then `PUT /customers/{id}` with `name: "Acme Ltd"` → the customer signs in and `GET /portal/me` shows `displayName: "Acme Ltd"`.
3. **Own name kept:** a contact self-registers with the email of an existing "Acme Ltd" contact, under the name "Sara". After another staff rename, Sara's profile still shows "Sara".
4. **External:** `PUT /api/v1/external/customers/erp/{externalId}` (API key with the customers-write scope) with a new name → in-step accounts follow, as in step 2.
5. **Unchanged:** a staff update that changes only tags does not modify `customer_accounts` (no `updated_at` change).

---

## Done Criteria

- [x] After `PUT /customers/{id}` with a new name, `GET /portal/me` of an in-step account shows it.
- [x] The same holds for the external upsert of an existing customer.
- [x] An account whose name differs from the old customer name keeps its own name.
- [x] An unchanged name does not touch portal accounts.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user.**
