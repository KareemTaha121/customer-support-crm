# Story intake

- Folder: `.squad/stories/platform/upload-response-uploader-name/intake.md`

---

## Feature

- **Feature name (display):** Platform (attachments)
- **Feature slug (folder under `plans/`):** `platform`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-25`
- **Work item type:** `Bug`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `bug`, `attachments`

---

## Title

```
Upload responses include the uploader name
```

---

## Description

```
QA round 2 (N5). POST /tickets/{id}/attachments returned "uploadedByName": null, while
GET /tickets/{id}/attachments shows "QA Round2 Agent". All three upload handlers pass null:
- Features/Tickets/TicketMessageSlices.cs:122
- Features/Customers/CustomerDetailSlices.cs:260
- Features/CustomerPortal/PortalTicketSlices.cs:156
```

---

## Acceptance criteria

```
- [x] Staff ticket, customer and portal uploads return the uploader name (staff display name / customer name).
- [x] dotnet build 0 warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** none
- **Depends on code areas or other stories:** found in the round 2 manual QA pass, [../../../qa/2026-10-01-manual-qa-report-round2.md](../../../qa/2026-10-01-manual-qa-report-round2.md).

## Technical hints (optional)

- `AttachmentService.ListAsync` already resolves the name (users, then customers with query filters ignored).

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/ or e2e/.
