# Story 10 — Customers UI (Story: FE-03)

## Prerequisites

- Story 08 completed: [08-story-core-platform-shell.md](08-story-core-platform-shell.md) (`ApiService`, `ApiError`, i18n, permissions, shared states/dialogs/form helpers).
- Story 09 completed: [09-story-authentication-staff-and-portal.md](09-story-authentication-staff-and-portal.md) (staff session and permission set).

---

## Story Goal

1. Staff find customers at `/customers` by name, company, number, email or phone, with server paging, sorting and filters (status, type, branch, department, tag).
2. Staff create a customer (with contacts) and are warned about duplicates before saving; edit the profile; soft delete with confirmation.
3. The details page `/customers/:id` shows tabs **Profile**, **Contacts** (add/edit/remove/set primary), **Notes** (internal, CRUD, pin), **Attachments** (upload/download/delete) and **History** (paged timeline), plus portal access grant/revoke.
4. Every action is hidden without its permission; server validation errors land on the matching field; full en/ar with RTL.

Not in scope: the customer's ticket list (Story 11 owns tickets), customer merge (no endpoint).

---

## Context — Read These Files First

1. `customer-support-crm-api/src/CustomerSupportCrm.Contracts/Customers/CustomerContracts.cs` (lines 1–100) — `CreateCustomerRequest`, `UpdateCustomerRequest`, `CustomerContactRequest`, `CustomerListItemResponse`, `CustomerResponse` (+ `CustomerStatsResponse`), `DuplicateCandidateResponse`, `CustomerNoteRequest/Response`, `CustomerActivityResponse`.
2. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Customers/CustomerProfileSlices.cs`
   - lines 37–53 create validator (name ≤ 200, language `en|ar`, branch required, ≤ 20 tags, ≤ 50 contacts); lines 85–95 `DUPLICATE_CUSTOMER` (409) unless `ignoreDuplicates`.
   - lines 135–147 update validator (adds `status` Active|Inactive); lines 178–195 delete refused with `CUSTOMER_HAS_OPEN_TICKETS`.
   - lines 207–233 list query: `page`, `pageSize` (≤ 100), `search`, `status`, `type`, `branchId`, `departmentId`, `tag`, `sortBy` = `name|number|createdAt|updatedAt`, `sortDirection`.
   - lines 324–386 endpoints: `GET /customers`, `GET /customers/duplicates?email&phone&excludeCustomerId`, `GET/PUT/DELETE /customers/{id}`, `POST /customers`.
3. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Customers/CustomerDetailSlices.cs`
   - lines 25–110 contacts (edit cannot change `type`; duplicate contact → `DUPLICATE_CUSTOMER`); lines 114–216 notes (`canEdit` per note, `NOTE_EDIT_FORBIDDEN`); lines 218–294 attachments; lines 298–345 history (`types` filter).
   - lines 349–447 endpoints: `POST/PUT/DELETE /customers/{id}/contacts[/{contactId}]`, `POST .../contacts/{contactId}/primary` (all return `CustomerResponse`), `GET/POST/PUT/DELETE .../notes`, `GET/POST/DELETE .../attachments`, `GET .../attachments/{attachmentId}` (binary), `GET .../history`.
4. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Customers/Common/CustomerQueries.cs` (lines 13–20) — error codes.
5. `customer-support-crm-api/src/CustomerSupportCrm.Domain/Customers/CustomerRecords.cs` (lines 116–128) — activity types (`customer.created`, `note.added`, `ticket.created`, ...); `Customer.cs` lines 6–36, 247–285 — enums, limits, contact value normalization (E.164 phones).
6. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/CustomerPortal/PortalAccountSlices.cs` (lines 315–391) — `POST/DELETE /customers/{id}/portal-access` (`customers.update`), `GrantPortalAccessRequest(email, password)` in `Contracts/Portal/PortalContracts.cs:53`, password 12–128, `PORTAL_ACCOUNT_EXISTS`.
7. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Branches/BranchEndpoints.cs` (lines 140–146) — `GET /branches` readable by any staff; `BranchResponse` includes `departments` (`Contracts/Organization/OrganizationContracts.cs:25–34`) → branch/department pickers.
8. Web: `core/http/api.service.ts`, `shared/form-errors.ts`, `shared/confirm-dialog.component.ts`, `features/auth/profile.page.ts` (style reference).

---

## Frontend Tasks

All files under `customer-support-crm-web/src/app/features/customers/`.

### 1 — Models and API

- `customers.models.ts` — interfaces mirroring the records above, enum unions (`CustomerType`, `CustomerStatus`, `ContactType`), `CustomerListQuery`, `CustomerErrorCodes`, `ACTIVITY_TYPES`.
- `customers.api.ts` — `CustomersApi` (`providedIn: 'root'`): `list`, `get`, `create`, `update`, `delete`, `findDuplicates`, `addContact`, `updateContact`, `removeContact`, `setPrimaryContact`, `listNotes`, `saveNote`, `deleteNote`, `listAttachments`, `uploadAttachment`, `downloadAttachment`, `deleteAttachment`, `history`, `grantPortalAccess`, `revokePortalAccess`, `branches`.

### 2 — Routes

- `customers.routes.ts` — `CUSTOMERS_ROUTES`: root `''` with `canActivate: [requirePermission(customers.view)]`, `resolve: { i18n: translationResolver('customers') }`, children `''` (list), `new` (form, `customers.create`), `:id` (details), `:id/edit` (form, `customers.update`).

### 3 — Pages and components

- `customer-list.page.ts` — toolbar (debounced search, status, type, branch, department, tag), `mat-table` + `mat-sort` (name, number, createdAt) + `mat-paginator`; row click → details; "New customer" (`customers.create`).
- `customer-form.page.ts` — create/edit form (type, name, company, language, status on edit, branch, department filtered by branch, tags chip input). Create mode adds a contacts `FormArray`; email/phone contacts trigger a debounced `GET /customers/duplicates` and show a warning panel linking the matches. On `DUPLICATE_CUSTOMER` a confirm dialog offers "Create anyway" (`ignoreDuplicates: true`). Server field errors via `{ silent: true }` + `applyServerErrors`.
- `customer-details.page.ts` — header (number, status pill, portal badge), actions Edit / Delete / Portal access, `mat-tab-group` hosting the tab components; keeps the `CustomerResponse` signal updated from contact actions.
- `customer-profile-tab.component.ts` — read-only profile and stats.
- `customer-contacts.component.ts` + `contact-dialog.component.ts` — list, add/edit (type locked on edit), remove (confirm), set primary.
- `customer-notes.component.ts` — composer + paged list, pin, edit inline, delete (only where `canEdit`).
- `customer-attachments.component.ts` — upload (file input), download (`ApiService.download` + `saveBlob`), delete.
- `customer-history.component.ts` — type filter, paged timeline.
- `portal-access-dialog.component.ts` — email (prefilled with primary email) + password (12–128).
- `public/i18n/customers/en.json`, `ar.json` — identical key sets.

---

## Edge Cases & Failure Modes

- `CUSTOMER_NOT_FOUND` (or out of the user's branch scope) on details → error state with retry and back link.
- `CUSTOMER_HAS_OPEN_TICKETS` on delete → server message in the snackbar; customer stays.
- `DUPLICATE_CUSTOMER` on create → confirm and resend with `ignoreDuplicates`; on contact save → error on the value field (no override exists server-side).
- `NOTE_EDIT_FORBIDDEN` → buttons hidden when `canEdit` is false; server message otherwise.
- Phone not E.164 → domain error surfaces as snackbar/field error; hint text shows the expected format.
- `PORTAL_ACCOUNT_EXISTS` → error on the email field in the portal dialog.
- Attachment too large / blocked type (413/400) → snackbar via the global handler.
- Changing branch clears a department that does not belong to it.

## Test Plan

Out of scope (build-level verification only). Manual smoke: search by phone, create with duplicate email → warning → create anyway, add/edit/remove contact, note CRUD, upload + download + delete file, history paging, grant + revoke portal access, delete a customer with and without open tickets, switch to Arabic.

## Verification Steps

1. **Frontend builds:** `cd customer-support-crm-web && npx ng build` with no errors or warnings in `features/customers/**`.

## Done Criteria

- [ ] All customer endpoints in `Application/Features/Customers` (and staff portal-access endpoints) are reachable from the UI.
- [ ] Server validation errors show on the right field; duplicates are warned before save.
- [ ] Actions are hidden without the matching permission; en/ar + RTL.
