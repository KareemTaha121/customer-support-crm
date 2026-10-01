# Story intake

- Folder: `.squad/stories/customer-management/customer-profiles-and-contacts/intake.md`

---

## Feature

- **Feature name (display):** Customer Management
- **Feature slug (folder under `plans/`):** `customer-management`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `CU-01`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `customers`

---

## Title

```
Customer profiles, search, duplicates and contacts
```

---

## Description

```
Staff manage customers (individuals and companies) through /api/v1/customers in
customer-support-crm-api. Slices live in two files under
Application/Features/Customers/ (CustomerProfileSlices.cs, CustomerDetailSlices.cs)
with shared helpers in Features/Customers/Common/CustomerQueries.cs. Contracts in
Contracts/Customers/CustomerContracts.cs.

Endpoints and permissions:
- GET    /customers                          customers.view    List: page, pageSize (max 100), search (name,
                                                               company, exact number, any contact value; phone
                                                               digits >= 6), status (Active|Inactive), type
                                                               (Individual|Company), branchId, departmentId, tag,
                                                               sortBy (name|number|createdAt|updatedAt), sortDirection
- GET    /customers/duplicates               customers.view    ?email=&phone=&excludeCustomerId= (organization-wide)
- GET    /customers/{id}                     customers.view    Profile + contacts + stats + hasPortalAccount
- POST   /customers                          customers.create  {type, name, companyName, preferredLanguage (en|ar),
                                                               branchId, departmentId, tags[], contacts[], ignoreDuplicates}
- PUT    /customers/{id}                     customers.update  {type, name, companyName, preferredLanguage, status,
                                                               branchId, departmentId, tags[]}
- DELETE /customers/{id}                     customers.delete  Soft delete
- POST   /customers/{id}/contacts            customers.update  {type, value, label, isPrimary}
- PUT    /customers/{id}/contacts/{cid}      customers.update  Update value/label (+ make primary)
- DELETE /customers/{id}/contacts/{cid}      customers.update  Remove (next contact of the type becomes primary)
- POST   /customers/{id}/contacts/{cid}/primary customers.update Set primary within the contact type

Rules:
- Number C-000001 from the customer_numbers Postgres sequence.
- Contacts validated per type: Email (EmailAddress), Phone/WhatsApp E.164
  (+[1-9] 7-15 digits; spaces/dashes removed, leading 00 -> +), Address/Other trimmed text.
- Same type+value twice on one customer -> 422 DUPLICATE_CONTACT.
- Email/phone already used by another customer -> 409 DUPLICATE_CUSTOMER
  (create can skip with ignoreDuplicates; contact save always checks).
- Branch/department scope: out-of-scope reads -> 404 CUSTOMER_NOT_FOUND; assigning to a
  unit outside the caller's scope -> 403 OUT_OF_SCOPE; inactive/unknown unit -> 404
  BRANCH_NOT_FOUND / DEPARTMENT_NOT_FOUND.
- Delete refused while the customer has non-resolved/closed tickets -> 409
  CUSTOMER_HAS_OPEN_TICKETS.
- Create/update/delete/contact changes write audit entries (customers.*).
- Messages localized (en/ar) in Messages.resx / Messages.ar.resx.
```

---

## Acceptance criteria

```
- [ ] Customer CRUD with soft delete and audit trail.
- [ ] Agents can search customers by name, phone, email or customer number.
- [ ] Duplicate detection on phone/email within the organization.
- [ ] Contacts are validated per type (E.164 phone, valid email); multiple contacts with a primary per type.
- [ ] Every record carries Branch / Department ownership and access is enforced.
- [ ] Arabic/English messages.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** P2-01..P2-07 (identity, permissions, audit), Phase 3 organization context (branches, departments, access scopes).
- **Depends on code areas or other stories:** `IAccessScopeProvider` / `AccessScope`, `IAuditTrail`, `ApiResults`, `PagedResult`, `EmailAddress`, ticket domain groundwork (open-ticket counts).

## Extra notes (optional)

- As-built intake written after `customer-support-crm-api` commit `0fd694e`.

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- Notes, attachments and interaction history (CU-02).
- Portal account grant/revoke (Customer Portal feature 08).
