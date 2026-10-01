# Story intake

- Folder: `.squad/stories/customer-management/staff-rename-portal-account-sync/intake.md`

---

## Feature

- **Feature name (display):** Customer Management
- **Feature slug (folder under `plans/`):** `customer-management`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-10`
- **Work item type:** `Bug`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `bug`, `customers`, `portal`

---

## Title

```
Customer renames by staff or integrations reach the portal account name
```

---

## Description

```
The portal profile (GET /portal/me) and new customer tokens read
CustomerAccount.DisplayName, not Customer.Name. BUG-04 (plan 40) keeps them in
step for PUT /portal/me only. Two other writes rename the customer and leave the
portal account on the old name:

- PUT /customers/{id}                              (UpdateCustomerHandler,
  Features/Customers/CustomerProfileSlices.cs ~148)
- PUT /external/customers/{system}/{externalId}    (external upsert,
  Features/Integrations/IntegrationSlices.cs ~356)

A customer can have several portal accounts (e.g. a company with several
contacts): the email is unique per account, and self-registration links to an
existing customer by email while keeping the registrant's own name
(CustomerAccount.Register(..., input.Name, ...)). Overwriting every account
would replace those personal names with the company name.

Fix:
- One shared helper renames, in the same SaveChanges, the customer's portal
  accounts whose DisplayName equals the customer's previous name (the accounts
  that were in step with it), using CustomerAccount.Rename (BUG-04).
- Call it from the staff update and the external upsert when the name changes.
- Accounts with their own (different) name are left unchanged.
```

---

## Acceptance criteria

```
- [ ] After PUT /customers/{id} with a new name, GET /portal/me of an in-step account shows it.
- [ ] The same for the external upsert of an existing customer.
- [ ] An account whose name differs from the old customer name keeps its own name.
- [ ] Unchanged names do not touch portal accounts.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** BUG-04 (`CustomerAccount.Rename`, plan 40).
- **Depends on code areas or other stories:** found while fixing BUG-04 (see `.squad/HANDOFF.md`, "Probable backend bugs").

## Technical hints (optional)

- Repo root: `customer-support-crm-api/` (branch `develop`). .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- Names inside already-issued customer tokens (they refresh on next sign-in, ≤ 8 h).
