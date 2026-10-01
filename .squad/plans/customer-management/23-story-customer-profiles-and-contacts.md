# Story 23 — Customer profiles, search, duplicates and contacts (Story: CU-01)

> As-built plan: written after implementation in `customer-support-crm-api` commit `0fd694e`; paths and line numbers refer to that commit unless a line says otherwise. Localization keys were added later in `0936711`; the database migration was generated later in `678ea67`.

## Prerequisites

- Phase 2 (Security & Administration, `0f87e2d`): `ICurrentUser`, permission policies, `IAuditTrail`, `EmailAddress`, `ApiResults`, `PagedResult`, `CommonRules`.
- Phase 3 (platform and organization context, `65c74a3`): `Branch`, `Department`, `IScopedEntity`, `AccessScope` / `IAccessScopeProvider`, `ISoftDeletable` handling in `AuditableEntityInterceptor`.
- Ticket domain groundwork (`Ticket`, `TicketStatus`) shipped in the same commit, so the customer stats can count tickets. Ticket slices come in feature 02 (`57e52f8`).
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

Staff keep one record per customer under `/api/v1/customers`:

1. Create, view, update and soft-delete a customer (individual or company) owned by a branch and an optional department.
2. Search, filter, sort and page customers, scoped to the caller's branches and departments.
3. Detect duplicates on email and phone across the organization.
4. Manage several typed contacts per customer, with one primary per contact type.

**Deviations from the intake (the code is authoritative):**

| Intake / feature spec | As built in `0fd694e` |
|---|---|
| One folder per slice (`Features/Customers/CreateCustomer/`, …) | Two files hold all slices: `CustomerProfileSlices.cs` (profile, list, duplicates) and `CustomerDetailSlices.cs` (contacts, notes, attachments, history), plus `Common/CustomerQueries.cs` |
| Separate `Contacts/Add` and `Contacts/Update` slices | One `SaveCustomerContactCommand` (`ContactId == null` means add). On update `Type` is required by the validator but ignored, so a contact cannot change type |
| Permissions `Customers.View`, `Customers.Notes.Manage`, … | Lower-case codes `customers.view`, `customers.create`, `customers.update`, `customers.delete`, `customers.notes_manage`, `customers.attachments_manage` (`Permissions.cs` lines 17–22) |
| Value objects `CustomerNumber`, `Address` | `Number` is a string (`Customer.FormatNumber`, `C-000042`); addresses are free text (`ContactType.Address`). Only `PhoneNumber` was added (`Domain/Shared/PhoneNumber.cs`) |
| Organization / Branch / Department ownership | `BranchId` (required) and `DepartmentId` (optional) only; there is no `OrganizationId` (single-organization deployment) |
| Duplicate detection within the organization | Organization-wide, ignoring branch scope on purpose (`CustomerQueries.cs` lines 118–121). `GET /customers/duplicates` therefore returns id, number and name of customers the caller may not otherwise see |
| Full audit trail | Create, update, delete, contact save and contact remove are audited. `SetPrimary` is not |
| Database migration in the story | No migration in `0fd694e`; the tables appear in `20260930104602_AddSupportOperations.cs` (`678ea67`) |

**Not in scope:** notes, attachments, history (story 24); portal access grant/revoke (feature 08, `00d3f35`). **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Application/Common/Authorization/AccessScope.cs` — lines 14–28 (`AccessScope`, `Everything`, `CanAccess`), 36–70 (`AccessScopeProvider`; `data.all_branches` → everything), 75–87 (`WhereInScope`), 92–99 (`EnsureAccess` → 404, existence not disclosed), 102–109 (`EnsureCanAssign` → 403 `OUT_OF_SCOPE`), 112–121 (`OrganizationErrors`).
2. `src/CustomerSupportCrm.Domain/Roles/Permissions.cs` — lines 17–22 (customer codes), 49 (`DataAllBranches`), 68 (agent defaults include every customer permission except delete), 77 (`CustomersDelete` for managers).
3. `src/CustomerSupportCrm.Domain/Shared/EmailAddress.cs` (Phase 2) — `Create` lower-cases and trims; `Normalize` is used by the duplicates query.
4. `src/CustomerSupportCrm.Infrastructure/Persistence/Interceptors/AuditableEntityInterceptor.cs` — lines 38–44: `Remove` of an `ISoftDeletable` becomes an update setting `IsDeleted`, `DeletedAt`, `DeletedBy`. `ApplicationDbContext.cs` line 72 / 81 adds the `IsDeleted` query filter.
5. `src/CustomerSupportCrm.Application/Common/Pagination/PagedResult.cs`, `Common/Validation/CommonRules.cs` (Phase 2) — `ToPagedResultAsync`, `ValidPage`, `ValidPageSize`, `OneOf`, `ValidSortDirection`.

---

## Backend Tasks

### 1 — Domain

Create file: `src/CustomerSupportCrm.Domain/Shared/PhoneNumber.cs` (lines 10–47) — `sealed partial record PhoneNumber`: `MaxLength = 16`, `InvalidCode = "INVALID_PHONE_NUMBER"`, `Create` (throws `DomainException`), `TryCreate`, `Normalize` (keeps digits and `+`, `00` → `+`), regex `^\+[1-9]\d{6,14}$`.

Create file: `src/CustomerSupportCrm.Domain/Customers/Customer.cs`:

- Enums `CustomerType { Individual, Company }`, `CustomerStatus { Active, Inactive }`, `ContactType { Email, Phone, WhatsApp, Address, Other }` (lines 6–25).
- `Customer : AggregateRoot<Guid>, IAuditableEntity, ISoftDeletable, IScopedEntity` (31–243). Constants `NameMaxLength = 200`, `MaxTags = 20`, `TagMaxLength = 50`; codes `INVALID_CUSTOMER`, `CONTACT_NOT_FOUND`, `DUPLICATE_CONTACT` (38–40). Denormalized `PrimaryEmail` / `PrimaryPhone` (77–81), `ExternalSystem` / `ExternalId` (84–86).
- `Create(...)` (110–127) raises `CustomerCreatedDomainEvent`; `UpdateProfile` (129–153) trims, checks language `en|ar`, de-duplicates tags case-insensitively; `MoveTo` (155–164); `SetStatus`; `LinkExternal` (168–174, used by the external API later).
- Contacts: `AddContact` (176–194 — first contact of a type becomes primary), `UpdateContact` (196–207), `RemoveContact` (209–220 — promotes the next contact of the type), `SetPrimary` (222–231), `RefreshPrimaries` (237–242 — phone falls back to WhatsApp).
- `CustomerContact` (245–300): `ValueMaxLength = 500`, `LabelMaxLength = 100`; `NormalizeValue` (278–285) validates per type.

### 2 — Persistence

- `src/CustomerSupportCrm.Application/Abstractions/Persistence/IApplicationDbContext.Customers.cs` — `Customers`, `CustomerNotes`, `CustomerActivities`, `Attachments` DbSets (Infrastructure twin `ApplicationDbContext.Customers.cs`).
- `src/CustomerSupportCrm.Application/Abstractions/Persistence/ISequenceGenerator.cs` — `NextValueAsync`; `Sequences.CustomerNumbers = "customer_numbers"`. Implementation `src/CustomerSupportCrm.Infrastructure/Persistence/PostgresSequenceGenerator.cs` (`SELECT nextval(...)`), registered in `Infrastructure/DependencyInjection.cs` line 64.
- `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/CustomerConfiguration.cs` — `customers` (lines 11–45: unique `Number`, indexes on `Name`, `PrimaryEmail`, `PrimaryPhone`, `(BranchId, DepartmentId)`, filtered unique `(ExternalSystem, ExternalId)`, `Tags` as `text[]`, restrict FKs to branch/department, `xmin` concurrency) and `customer_contacts` (47–59, index `(Type, Value)` used by duplicate detection). `SequenceConfiguration.AddSequences` (106–113) is called from `ApplicationDbContext.cs` line 71.
- Migration: none in this commit (see Deviations).

### 3 — Contracts

Create file: `src/CustomerSupportCrm.Contracts/Customers/CustomerContracts.cs` — `CustomerContactRequest` (7), `CreateCustomerRequest` (11–20, `IgnoreDuplicates = false`), `UpdateCustomerRequest` (23–31), `CustomerContactResponse` (33), `CustomerListItemResponse` (35–50), `CustomerStatsResponse` (52: open/total tickets, last interaction, average satisfaction), `CustomerResponse` (54–73, includes `HasPortalAccount`), `DuplicateCandidateResponse` (76).

### 4 — Shared queries

Create file: `src/CustomerSupportCrm.Application/Features/Customers/Common/CustomerQueries.cs`:

- `CustomerErrors` (13–20): `CUSTOMER_NOT_FOUND`, `DUPLICATE_CUSTOMER`, `NOTE_NOT_FOUND`, `NOTE_EDIT_FORBIDDEN`, `CUSTOMER_HAS_OPEN_TICKETS`.
- `LoadAsync` (27–33, tracked with contacts + `EnsureAccess`), `EnsureAccessibleAsync` (36–43, `AnyAsync` with `WhereInScope`).
- `ProjectListItems` (45–61) and `GetResponseAsync` (63–116): `AsNoTracking` projections with correlated sub-selects for branch/department names, ticket counts, last activity, average satisfaction and portal account.
- `FindDuplicatesAsync` (122–154, organization-wide, max 20 matches) and `MatchableValues` (156–162, Email vs Phone/WhatsApp).

### 5 — Profile slices

Create file: `src/CustomerSupportCrm.Application/Features/Customers/CustomerProfileSlices.cs`:

- **Create** (26–103): validator (37–54) — `Type` enum name, `Name` ≤ 200, language `en|ar` (`INVALID_VALUE`), `BranchId` required, ≤ 20 tags, ≤ 50 contacts. Handler: `EnsureCanAssign` → `OrganizationUnits.EnsureValidAsync` (105–120, active branch, department in branch) → next sequence number → `Customer.Create` → add contacts → duplicate check unless `IgnoreDuplicates` (85–95, 409 `DUPLICATE_CUSTOMER`) → audit `customers.created` → save → `GetResponseAsync(AccessScope.Everything)`.
- **Update** (124–174): moving branch/department re-checks scope and units; audit `customers.updated` with before/after; timeline `customer.updated`.
- **Delete** (176–195): 409 `CUSTOMER_HAS_OPEN_TICKETS` while a ticket is not `Resolved`/`Closed`; `db.Customers.Remove` (soft delete); audit `customers.deleted`.
- **Get** (199–205), **List** (209–296): validator 221–233; search (243–255) on upper-cased name, company, exact number, any contact value, plus normalized phone digits when ≥ 6 chars (`#pragma warning disable CA1304, CA1311, CA1862`); filters; sort switch then `ThenBy(Id)`.
- **Find duplicates** (298–309): normalizes the email, accepts the phone only when it parses as E.164.
- `CustomerCreatedTimelineHandler` (313–320) writes `customer.created` (timeline belongs to story 24).
- `CustomerProfileEndpoints` (324–387): group `/customers`, tag `Customers`; names `ListCustomers`, `FindDuplicateCustomers`, `GetCustomer`, `CreateCustomer` (201 + `Location`), `UpdateCustomer`, `DeleteCustomer`.

### 6 — Contact slices

In `src/CustomerSupportCrm.Application/Features/Customers/CustomerDetailSlices.cs`:

- `SaveCustomerContactCommand` + validator (28–39) and handler (41–78): add or update, optional make-primary, then duplicate check against other customers (409 `DUPLICATE_CUSTOMER`), audit `customers.contact_saved`.
- `RemoveCustomerContactCommand` (80–95), audit `customers.contact_removed`. `SetPrimaryContactCommand` (97–110), not audited.
- Endpoints (355–377) on group `/customers/{customerId:guid}`: `AddCustomerContact`, `UpdateCustomerContact`, `RemoveCustomerContact`, `SetPrimaryCustomerContact`, all `customers.update`, all return the full `CustomerResponse`.

### 7 — DI and localization

- No DI line for the slices (assembly scanning). `CustomerTimeline` is registered in `Application/DependencyInjection.cs` line 34.
- `Messages.resx` (at `0936711`): `OUT_OF_SCOPE` (117), `CUSTOMER_NOT_FOUND` (138), `CUSTOMER_HAS_OPEN_TICKETS` (141), `CONTACT_NOT_FOUND` (144), `DUPLICATE_CONTACT` (147), `INVALID_PHONE_NUMBER` (150). `Messages.ar.resx` also has `DUPLICATE_CUSTOMER` (159) and `INVALID_CUSTOMER` (162); in English those codes keep the specific message from the throw site.
- `docs/endpoints.md` lines 71–89 (added in `fdd93cd`) list the routes.

---

## Edge Cases & Failure Modes

- **Out-of-scope customer** — every read/write returns 404 `CUSTOMER_NOT_FOUND`, not 403.
- **Create/move into a unit outside the caller's scope** — 403 `OUT_OF_SCOPE`; inactive or unknown branch → 404 `BRANCH_NOT_FOUND`; department not in branch → 404 `DEPARTMENT_NOT_FOUND`.
- **National phone format** (`0501234567`) — 422 `INVALID_PHONE_NUMBER`; `00966…` is accepted as `+966…`.
- **Same contact twice on one customer** — 422 `DUPLICATE_CONTACT`; on another customer — 409 `DUPLICATE_CUSTOMER` (create only; `ignoreDuplicates: true` skips it).
- **Removing the primary contact** — the next contact of that type becomes primary; `PrimaryEmail` / `PrimaryPhone` are refreshed.
- **Delete with open tickets** — 409 `CUSTOMER_HAS_OPEN_TICKETS`. A soft-deleted customer disappears from every query (global filter) but keeps its number.
- **Concurrent edits** — `xmin` token → 409 `CONFLICT`.
- **Number race** — the Postgres sequence makes numbers unique without locking; gaps are possible.

---

## Test Plan

Test projects are **out of scope** (`tests/` must not be modified). No tests are added, changed or removed by this story, and no test in the repo (at `2956767`) covers customers.

---

## Verification Steps

1. **Backend builds:** in `customer-support-crm-api/` run `dotnet build` — 0 warnings, 0 errors.
2. **Run:** apply migrations, start the API, sign in via `POST /api/v1/auth/login`, export `TOKEN`, and get a branch id from `GET /api/v1/branches`.
3. **Create:** `curl -i -X POST https://localhost:<port>/api/v1/customers -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"type":"Individual","name":"Sara Ali","preferredLanguage":"ar","branchId":"<id>","contacts":[{"type":"Phone","value":"00966 50 123 4567","isPrimary":true},{"type":"Email","value":"Sara@Example.com"}]}'` → 201, number `C-000001`, phone `+966501234567`, email lower-cased.
4. **Duplicate:** repeat the same call → 409 `DUPLICATE_CUSTOMER`; `GET …/customers/duplicates?phone=%2B966501234567` → one candidate; add `"ignoreDuplicates":true` → 201.
5. **List:** `GET …/customers?search=1234567&sortBy=number&sortDirection=desc&pageSize=10` → paged `meta`; `status=Archived` → 400 `INVALID_VALUE`.
6. **Contacts:** `POST …/customers/{id}/contacts` with `{"type":"Phone","value":"0501234567"}` → 422 `INVALID_PHONE_NUMBER`; `POST …/contacts/{cid}/primary` → `primaryPhone` changes.
7. **Delete:** `DELETE …/customers/{id}` → 200; `GET` the same id → 404 `CUSTOMER_NOT_FOUND`.
8. **Scope:** an agent limited to another branch gets 404 on `GET …/customers/{id}`.
9. **Localization:** a failing call with `Accept-Language: ar` → Arabic message.

---

## Done Criteria

- [x] Customer create, get, list, update and soft delete exist, use the standard envelope and enforce `customers.*` permissions.
- [x] Create, update and delete write audit entries; soft delete via `ISoftDeletable`.
- [x] Search by name, company, number, email and phone (digits) with filters, sorting and paging.
- [x] Duplicate detection on email/phone, organization-wide, on create and contact save, plus `GET /customers/duplicates`.
- [x] Contacts validated per type (E.164 phone, email), one primary per type.
- [x] Branch/department ownership enforced on every read and write. (No organization id — single organization.)
- [x] Error codes have Arabic messages (added in `0936711`).
- [ ] Contact type change on update — not supported (type is ignored).
- [ ] `SetPrimary` is not audited.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/` by this story.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to Story 24.**
