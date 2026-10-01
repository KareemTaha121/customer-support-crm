# Story intake

- Folder: `.squad/stories/customer-management/customer-notes-attachments-and-history/intake.md`

---

## Feature

- **Feature name (display):** Customer Management
- **Feature slug (folder under `plans/`):** `customer-management`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `CU-02`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `customers`

---

## Title

```
Customer notes, attachments and interaction history
```

---

## Description

```
Internal notes, file attachments and a paged interaction timeline on a customer, under
/api/v1/customers/{customerId}. Slices in Application/Features/Customers/CustomerDetailSlices.cs;
file handling in Application/Features/Attachments/AttachmentService.cs (shared by tickets and
knowledge base later); timeline writer CustomerTimeline in Features/Customers/Common/CustomerQueries.cs.

Endpoints and permissions:
- GET    /customers/{id}/notes                    customers.view                Paged (page, pageSize); pinned first, newest first
- POST   /customers/{id}/notes                    customers.notes_manage        {body (<= 10000), isPinned}
- PUT    /customers/{id}/notes/{noteId}           customers.notes_manage        Author only, or data.all_branches
- DELETE /customers/{id}/notes/{noteId}           customers.notes_manage        Author only, or data.all_branches
- GET    /customers/{id}/attachments              customers.view                List metadata (newest first)
- POST   /customers/{id}/attachments              customers.attachments_manage  multipart/form-data "file"
- GET    /customers/{id}/attachments/{aid}        customers.view                Download stream, X-Content-Type-Options: nosniff
- DELETE /customers/{id}/attachments/{aid}        customers.attachments_manage  Removes row, then file
- GET    /customers/{id}/history                  customers.view                Paged; types=comma-separated activity types

Rules:
- Every call first checks the customer exists and is in the caller's branch/department
  scope -> 404 CUSTOMER_NOT_FOUND.
- Notes are internal: no portal endpoint reads customer_notes. Unknown note -> 404
  NOTE_NOT_FOUND; editing someone else's note -> 403 NOTE_EDIT_FORBIDDEN; empty/too long ->
  400 validation (422 INVALID_NOTE from the domain).
- Uploads: max 20 MB, extension allow-list (pdf, png, jpg/jpeg, gif, webp, docx, xlsx, pptx,
  doc, xls, zip, txt, csv), magic-byte signature check, client MIME ignored (content type
  derived from extension), file name sanitized, storage key never contains user input.
  Errors: 400 FILE_EMPTY / FILE_TOO_LARGE / FILE_TYPE_NOT_ALLOWED / FILE_SIGNATURE_MISMATCH.
  Unknown attachment or missing file -> 404 ATTACHMENT_NOT_FOUND. DB stores metadata only.
- Customer attachments are stored with isPublic = false.
- Timeline: append-only customer_activities rows (customer.created, customer.updated,
  note.added, attachment.added; ticket.* and portal.sign_in are written by later features),
  read as an AsNoTracking projection, newest first.
- Attachment deletion is audited (customers.attachment_deleted).
```

---

## Acceptance criteria

```
- [ ] Interaction history is a paged, read-optimized projection (not full aggregate loads).
- [ ] Attachments stored in object storage; DB stores metadata only; size/MIME/extension/signature validated.
- [ ] Notes are internal-only and never exposed through the customer portal.
- [ ] Agents can add internal notes and upload attachments to a customer.
- [ ] Notes, attachments and history respect the customer's branch/department scope.
- [ ] Arabic/English messages.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** CU-01
- **Depends on code areas or other stories:** `IFileStorage` / `LocalFileStorage` and `FileUploadRules` (Phase 3), `IAccessScopeProvider`, `ICurrentUser`, `IAuditTrail`.

## Extra notes (optional)

- As-built intake written after `customer-support-crm-api` commit `0fd694e`.
- Portal access grant/revoke (`POST/DELETE /customers/{id}/portal-access`) does **not** live in this feature; it was added in `00d3f35` under `Features/CustomerPortal/PortalAccountSlices.cs` (feature 08).

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- Malware scanning, cloud object storage provider.
