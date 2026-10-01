# Customer Management

Feature spec: [../../features/01-customer-management.md](../../features/01-customer-management.md).
Repo: `customer-support-crm-api`. Out of scope for every story: Docker, deployment, CI/CD, and files under `tests/`.

| NN | Tracker id | Plan file | Title | Depends on | Status |
|----|-----------|-----------|-------|-----------|--------|
| 23 | CU-01 | [23-story-customer-profiles-and-contacts.md](23-story-customer-profiles-and-contacts.md) | Customer profiles, search, duplicates and contacts | Phase 2, Phase 3 | Done |
| 24 | CU-02 | [24-story-customer-notes-attachments-and-history.md](24-story-customer-notes-attachments-and-history.md) | Customer notes, attachments and interaction history | 23 | Done |
| 46 | BUG-10 | [46-story-staff-rename-portal-account-sync.md](46-story-staff-rename-portal-account-sync.md) | Customer renames by staff or integrations reach the portal account name | 23, 40 | Done (`dca992c`) |

Both stories were implemented in `customer-support-crm-api` commit `0fd694e` (feat: add customer management (feature 01)) without plans; these are **as-built** plans and their paths and line numbers refer to `0fd694e`. Later commits that touch the feature: `0936711` (en/ar messages for the error codes), `678ea67` (the migration `20260930104602_AddSupportOperations` that creates the customer tables), `fdd93cd` (`docs/endpoints.md`). `00d3f35` (feature 08) added `CustomerAccount.RestartVerification` and the staff portal-access endpoints. Nothing in `Features/Customers` or `Features/Attachments` has changed since `0fd694e`.

Frontend: [../frontend/10-story-customers-ui.md](../frontend/10-story-customers-ui.md).

Known gaps and drift:

- **Object storage** — attachments go through `IFileStorage`, but the only provider is `LocalFileStorage` (filesystem). No malware scanning.
- **Duplicate lookup crosses branch scope** on purpose; `GET /customers/duplicates` returns id, number and name of customers outside the caller's scope.
- **Audit coverage** — note add/edit/delete and set-primary-contact are not audited.
- **Timeline** — no call activity; `chat.started` is defined but never written.
- **Contact type** cannot be changed on update (the request type is ignored).
- **No organization id** on customers (single-organization model); ownership is branch + optional department.
- **Portal access** grant/revoke (`/customers/{id}/portal-access`) belongs to feature 08 (`Features/CustomerPortal/PortalAccountSlices.cs`), not to this feature.
- **No tests** cover customers, notes, attachments or `FileUploadRules`.
- Slices are grouped into two files instead of one folder per use case.
