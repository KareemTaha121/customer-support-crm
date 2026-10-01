# Story 24 — Customer notes, attachments and interaction history (Story: CU-02)

> As-built plan: written after implementation in `customer-support-crm-api` commit `0fd694e`; paths and line numbers refer to that commit unless a line says otherwise. Localization keys were added later in `0936711`; the database migration was generated later in `678ea67`.

## Prerequisites

- Story 23 completed: [23-story-customer-profiles-and-contacts.md](23-story-customer-profiles-and-contacts.md) — `Customer`, `CustomerQueries.EnsureAccessibleAsync`, `CustomerErrors`, DbSets.
- Phase 3 (`65c74a3`): `IFileStorage` (`Application/Abstractions/Files/IFileStorage.cs` lines 9–14), `LocalFileStorage`, `FileUploadRules`, `ICurrentUser.HasPermission`.
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

On a customer's record, staff can:

1. Write, edit, pin and delete internal notes (author-owned).
2. Upload, list, download and delete files; the database keeps metadata only.
3. Read one paged timeline of everything that happened with the customer.

**Deviations from the intake (the code is authoritative):**

| Intake / feature spec | As built in `0fd694e` |
|---|---|
| Attachments in object storage | Stored through `IFileStorage`, whose only implementation is `LocalFileStorage` (filesystem, default `App_Data/storage`, `Infrastructure/Files/LocalFileStorage.cs` lines 11–19). A cloud provider is not built |
| MIME validated | The client's MIME type is ignored; the content type is derived from the verified extension (`FileUploadRules.cs` lines 6–14). No malware scanning |
| Note edit/delete permission | Author only, or anyone with `data.all_branches` (not a dedicated permission) |
| Notes and attachments audited | Only attachment deletion is audited (`customers.attachment_deleted`). Note add/edit/delete are not audited; note add and attachment upload go to the timeline |
| Timeline covers calls | No call activity type; `chat.started` is defined (`CustomerRecords.cs` line 127) but nothing writes it. Ticket entries are written by feature 02 (`Tickets/TicketEventHandlers.cs`), `portal.sign_in` by feature 08 |
| Attachments paged | `GET /customers/{id}/attachments` returns the full list (no paging). `CustomerAttachmentsResponse` (`CustomerContracts.cs` line 100) is unused |
| Portal access grant/revoke | Not here: `POST/DELETE /customers/{id}/portal-access` live in `Features/CustomerPortal/PortalAccountSlices.cs` (lines 315–392 at `00d3f35`, feature 08), permission `customers.update` |

**Not in scope:** profile and contacts (story 23). **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Application/Features/Customers/Common/CustomerQueries.cs` — lines 36–43 `EnsureAccessibleAsync` (existence + scope, 404), 164–173 `ParseJson`, 177–190 `CustomerTimeline.Record` (adds a row to the current unit of work; actor = current user when authenticated).
2. `src/CustomerSupportCrm.Application/Common/Files/FileUploadRules.cs` (Phase 3) — lines 16–23 (20 MB limit, error codes), 34–54 (extension → content type + signatures, `Documents` set), 57–99 (`ValidateAsync`: empty, size, extension, magic bytes, text files must not contain NUL), 102–103 (`CreateStorageKey`, no user input), 105–115 (`Sanitize`), 117–118 (errors are `ValidationException` on field `File` → 400).
3. `src/CustomerSupportCrm.Infrastructure/Files/LocalFileStorage.cs` — lines 19–37; registered in `Infrastructure/DependencyInjection.cs` lines 75–76.
4. `src/CustomerSupportCrm.Domain/Roles/Permissions.cs` — lines 21–22 (`customers.notes_manage`, `customers.attachments_manage`), 49 (`data.all_branches`).

---

## Backend Tasks

### 1 — Domain

- `src/CustomerSupportCrm.Domain/Customers/CustomerRecords.cs`:
  - `CustomerNote : Entity<Guid>` (7–59): `BodyMaxLength = 10_000`, `INVALID_NOTE`, `Create(customerId, authorId, body, isPinned, now)`, `Edit(body, isPinned, now)`, trimmed body.
  - `CustomerActivity : Entity<Guid>` (65–114): append-only; `TypeMaxLength = 64`, `SummaryMaxLength = 500` (summary truncated), `Data` JSON string, `Record(...)` factory.
  - `CustomerActivityTypes` (116–128): `customer.created`, `customer.updated`, `note.added`, `attachment.added`, `ticket.created`, `ticket.status_changed`, `ticket.message`, `ticket.feedback`, `portal.sign_in`, `chat.started`.
- `src/CustomerSupportCrm.Domain/Attachments/Attachment.cs` — `AttachmentOwnerTypes` (6–11: `customer`, `ticket`, `kb_article`); `Attachment` (17–93): owner type/id, optional `ParentId`, `FileName`, `ContentType`, `Size`, `StorageKey`, `IsPublic`, `UploadedBy` / `UploadedByCustomerId`.
- `CustomerAccount` (`Domain/Customers/CustomerAccount.cs`) is portal groundwork only; here it feeds `HasPortalAccount`.

### 2 — Persistence

`src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/CustomerConfiguration.cs` — `customer_notes` (61–72, index `(CustomerId, CreatedAt)`, cascade), `customer_activities` (74–88, `Data` `jsonb`, index `(CustomerId, OccurredAt)` and `TicketId`, cascade), `attachments` (90–104, index `(OwnerType, OwnerId)` and `ParentId`, no FK because the owner is polymorphic).

### 3 — Contracts

- `src/CustomerSupportCrm.Contracts/Customers/CustomerContracts.cs` — `CustomerNoteRequest(Body, IsPinned = false)` (78), `CustomerNoteResponse` (80–88, `CanEdit`), `CustomerActivityResponse` (90–98, `Data` as `JsonElement?`).
- `src/CustomerSupportCrm.Contracts/Common/AttachmentResponse.cs` (4–12) — `DownloadUrl` is a relative API path needing the owner's authorization.

### 4 — Attachment service

Create file: `src/CustomerSupportCrm.Application/Features/Attachments/AttachmentService.cs` — `StoredFile` record (13) and `sealed class AttachmentService(IApplicationDbContext, IFileStorage, TimeProvider)` (19–103), registered in `Application/DependencyInjection.cs` line 33:

- `StoreAsync` (24–45): `FileUploadRules.ValidateAsync(..., MaxAttachmentBytes, Documents)` → storage key → `storage.SaveAsync` → `db.Attachments.Add` (caller saves).
- `GetAsync` (47–51): must match owner type/id (and `IsPublic` when `publicOnly`) → 404 `ATTACHMENT_NOT_FOUND`.
- `OpenAsync` (53–59), `DeleteAsync` (62–68: remove row, save, then delete file), `ListAsync` (70–90, uploader name from users or customers), `ToResponse` (92–93), `ToDownload` (96–102: `X-Content-Type-Options: nosniff`, `Results.Stream`).

### 5 — Notes slices

In `src/CustomerSupportCrm.Application/Features/Customers/CustomerDetailSlices.cs`:

- `ListCustomerNotesQuery` + handler (114–144): pinned first, newest first, author name sub-select, `CanEdit = data.all_branches || author == me`.
- `SaveCustomerNoteCommand` + validator + handler (147–195): add (timeline `note.added`, summary cut at 120 chars) or edit (404 `NOTE_NOT_FOUND`, 403 `NOTE_EDIT_FORBIDDEN`).
- `DeleteCustomerNoteCommand` + handler (197–216): same author rule; hard delete.

### 6 — Attachment slices

Same file: `ListCustomerAttachmentsQuery` (220–230), `UploadCustomerAttachmentCommand` (232–262; `isPublic: false`, timeline `attachment.added` with `{id, fileName}`), `DownloadCustomerAttachmentQuery` (264–275), `DeleteCustomerAttachmentCommand` (277–289; audit then `DeleteAsync`), `CustomerAttachmentPaths.Download` (291–294).

### 7 — History slice

Same file: `GetCustomerHistoryQuery` (299–300, `Types` comma-separated), validator (302–310, `Types` ≤ 500), handler (313–345): `AsNoTracking` on `CustomerActivities`, optional type filter, `OrderByDescending(OccurredAt).ThenByDescending(Id)`, actor name sub-select, paged.

### 8 — Endpoints

`CustomerDetailEndpoints` (349–448), group `/customers/{customerId:guid}`, tag `Customers`:

| Route | Permission | Name |
|---|---|---|
| `GET /notes` | `customers.view` | `ListCustomerNotes` |
| `POST /notes`, `PUT /notes/{noteId}`, `DELETE /notes/{noteId}` | `customers.notes_manage` | `AddCustomerNote`, `UpdateCustomerNote`, `DeleteCustomerNote` |
| `GET /attachments`, `GET /attachments/{attachmentId}` | `customers.view` | `ListCustomerAttachments`, `DownloadCustomerAttachment` |
| `POST /attachments` (`IFormFile file`, `.DisableAntiforgery()`) | `customers.attachments_manage` | `UploadCustomerAttachment` |
| `DELETE /attachments/{attachmentId}` | `customers.attachments_manage` | `DeleteCustomerAttachment` |
| `GET /history` | `customers.view` | `GetCustomerHistory` |

### 9 — Localization

`Messages.resx` (at `0936711`): `NOTE_NOT_FOUND` (153), `INVALID_NOTE` (156), `FILE_EMPTY` (159), `FILE_TYPE_NOT_ALLOWED` (162), `FILE_SIGNATURE_MISMATCH` (165). `Messages.ar.resx` also has `NOTE_EDIT_FORBIDDEN` (177), `ATTACHMENT_NOT_FOUND` (183), `FILE_TOO_LARGE` (189).

---

## Edge Cases & Failure Modes

- **Customer out of scope or deleted** — 404 `CUSTOMER_NOT_FOUND` on every note/attachment/history call.
- **Attachment of another customer** — `GetAsync` matches owner id → 404 `ATTACHMENT_NOT_FOUND`; row present but file missing → same code.
- **Renamed executable** (`evil.exe` → `evil.pdf`) — 400 `FILE_SIGNATURE_MISMATCH`; `.exe` → `FILE_TYPE_NOT_ALLOWED`; 0 bytes → `FILE_EMPTY`; > 20 MB → `FILE_TOO_LARGE`. The Kestrel body limit (`SecurityExtensions.cs` line 190 at `2956767`) also applies.
- **Path tricks in the file name** — `Sanitize` keeps only the file part and strips quotes/slashes; the storage key is generated.
- **Delete ordering** — the row is removed and saved before the file is deleted, so a failed file delete leaves an orphan file, never a broken row.
- **Note by a colleague** — 403 `NOTE_EDIT_FORBIDDEN` unless `data.all_branches`.
- **Portal exposure** — no portal slice reads `customer_notes`; customer attachments are `IsPublic = false`.
- **Unknown history types** — simply match nothing (empty page).

---

## Test Plan

Test projects are **out of scope** (`tests/` must not be modified). No tests are added, changed or removed by this story, and no test in the repo (at `2956767`) covers customer notes, attachments, `FileUploadRules` or history.

---

## Verification Steps

1. **Backend builds:** `dotnet build` — 0 warnings, 0 errors.
2. **Note:** `curl -X POST https://localhost:<port>/api/v1/customers/{id}/notes -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"body":"VIP, call before noon","isPinned":true}'` → 200; `GET …/notes` → pinned first, `canEdit: true`. Edit it with another agent's token → 403 `NOTE_EDIT_FORBIDDEN`.
3. **Upload:** `curl -X POST …/customers/{id}/attachments -H "Authorization: Bearer $TOKEN" -F "file=@contract.pdf"` → 200 with `downloadUrl`; a `.txt` renamed to `.pdf` → 400 `FILE_SIGNATURE_MISMATCH`.
4. **Download:** `curl -i …/customers/{id}/attachments/{aid}` → `Content-Type: application/pdf`, `X-Content-Type-Options: nosniff`, attachment file name.
5. **Delete:** `DELETE …/attachments/{aid}` → 200; download again → 404 `ATTACHMENT_NOT_FOUND`.
6. **History:** `GET …/customers/{id}/history?types=note.added,attachment.added&pageSize=5` → paged, newest first.
7. **Localization:** a failing call with `Accept-Language: ar` → Arabic message.

---

## Done Criteria

- [x] Notes: list, add, edit, pin, delete; internal only; author rule enforced.
- [x] Attachments: upload, list, download, delete; DB stores metadata only; size, extension and signature validated; content type derived server-side.
- [ ] Object storage — only the local filesystem provider exists (behind `IFileStorage`).
- [x] Interaction history is a paged `AsNoTracking` projection over the denormalized `customer_activities` table.
- [ ] Timeline covers calls and chats — no writer for `chat.started`, no call type.
- [x] Every call checks customer existence and branch/department scope.
- [ ] Note changes audited — only attachment deletion is audited.
- [x] Error codes have Arabic messages (added in `0936711`).
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/` by this story.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding.**
