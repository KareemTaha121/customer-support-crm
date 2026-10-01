# Story 61 — Upload responses include the uploader name (Bug: BUG-25)

> Fix plan, implemented in `customer-support-crm-api` commit `91ec854` (`develop`). Paths and line numbers refer to that commit.
> Intake: [../../stories/platform/upload-response-uploader-name/intake.md](../../stories/platform/upload-response-uploader-name/intake.md)

## Prerequisites

- Story 20 ([20-story-backend-foundation.md](20-story-backend-foundation.md)): `AttachmentService` and `IFileStorage`.
- Stories 24 (customer attachments), 26 (ticket attachments) and 31 (portal attachments): the three upload handlers.

---

## Story Goal

`AttachmentService.ListAsync` resolves `UploadedByName` from the staff user (`UploadedBy`) or, for portal uploads, from the customer (`UploadedByCustomerId`, with query filters ignored). The three upload endpoints instead built their response with `AttachmentService.ToResponse(attachment, null, …)`, so the response right after an upload always had `"uploadedByName": null`.

| Endpoint | Before | After |
|---|---|---|
| `POST /tickets/{id}/attachments` | `null` | staff display name |
| `POST /customers/{id}/attachments` | `null` | staff display name |
| `POST /portal/tickets/{id}/attachments` | `null` | customer name |

**Deviation from the intake:** none.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Application/Features/Attachments/AttachmentService.cs`: `ListAsync` (70–90) and the new `ToResponseAsync` (92–102).
2. `Features/Tickets/TicketMessageSlices.cs:122`, `Features/Customers/CustomerDetailSlices.cs:260`, `Features/CustomerPortal/PortalTicketSlices.cs:156`: the three callers.

---

## Backend Tasks

1. Replace the static `ToResponse(Attachment, string?, string)` with an instance method:
   ```csharp
   public async Task<AttachmentResponse> ToResponseAsync(Attachment a, string downloadUrl, CancellationToken cancellationToken)
   ```
   It resolves the name the same way as `ListAsync`: the user's `DisplayName` when `UploadedBy` is set, otherwise the customer's `Name` with `IgnoreQueryFilters()` so soft-deleted customers still resolve, otherwise `null`.
2. Each handler already injects `AttachmentService attachments`, so each calls `await attachments.ToResponseAsync(attachment, <download path>, cancellationToken)` after `SaveChangesAsync`.

No contract, migration or frontend change.

---

## Test Plan

Test projects are out of scope.

---

## Verification Steps

1. `dotnet build … -o <temp>`: 0 warnings, 0 errors.
2. As admin, upload `n5.txt` to T-000004: `uploadedByName` is "Administrator".
3. As the portal customer, upload to the same ticket: "QA Test Customer Renamed".
4. As admin, upload to customer C-000005: "Administrator".

All passed on 2026-10-01.

---

## Done Criteria

- [x] All three upload endpoints return the uploader name.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user.**
