# Story 26 — Ticket conversation, attachments, categories and event handlers (Story: TK-02)

> As-built plan: written after implementation. `TicketMessage`, `TicketCategory` and `TicketHistory` were committed with the ticket domain in `customer-support-crm-api` `0fd694e`; the message/attachment/category slices, event handlers and `Attachment.AttachTo` in `57e52f8`; the schema in migration `20260930104602_AddSupportOperations` (`678ea67`); default categories seeded in `b81ba28`. Paths and line numbers refer to `develop` HEAD `2956767` (the files below are unchanged since `57e52f8`, except where noted).

## Prerequisites

- Story 25 completed: [25-story-ticket-lifecycle.md](25-story-ticket-lifecycle.md) — `Ticket`, `TicketQueries`, `TicketErrors`, `TicketHistoryRecorder`.
- Platform phase 3 (`65c74a3`) / Customer Management (`0fd694e`) — `AttachmentService`, `FileUploadRules`, `Attachment`, `AttachmentOwnerTypes`, `NotificationSender`, `CustomerTimeline`, `ICurrentCustomer`.
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

1. Agents read the full conversation and add public replies or internal notes, with @mentions and files.
2. Files are uploaded to the ticket first, then linked to the message they are sent with; they can be listed, downloaded and deleted.
3. Admins maintain hierarchical ticket categories with routing defaults (department, priority) and Arabic names.
4. Domain-event handlers write history, customer-timeline entries and in-app notifications for messages, SLA events and feedback.

**Deviations from the intake (the code is authoritative):**

| Intake (feature spec 02) | As built |
|---|---|
| `Messages/` slices `AddReply`, `AddInternalNote`, `List` | One `POST /tickets/{id}/messages` with `isInternal`; list is `GET /tickets/{id}/messages` (in `TicketQuerySlices.cs`) |
| `TicketAttachment` entity | Shared `Attachment` (`OwnerType = "ticket"`, `OwnerId = ticketId`, `ParentId = messageId`) |
| `TicketPriority` entity + `Priorities/` CRUD | `TicketPriority` is a fixed enum (Low, Medium, High, Urgent); no priority endpoints |
| `Categories/` CRUD | List, create, update only; no delete (`TicketErrors.CategoryInUse` / `CATEGORY_IN_USE` is declared and localized but unused); deactivate via `isActive` |
| History of messages | Messages do **not** write `ticket_history` rows; they produce customer-timeline entries (public only) and notifications |
| `Tickets.Categories.Manage` | `tickets.categories_manage`; `GET /ticket-categories` needs only authentication |

**Not in scope:** delivery of public replies on email/WhatsApp/SMS/chat (feature 03 `ChannelDeliveryHandlers`), portal messages/feedback (feature 08), `GET /kb/suggestions?ticketId=` (feature 06; `889dfcf` switched it to any-term matching). **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Domain/Tickets/TicketRecords.cs` — `TicketMessage` (7–82: body ≤ 50 000, `INVALID_MESSAGE`, internal notes only by agents at 63–66, `MentionedUserIds`, `ExternalMessageId`), `TicketCategory` (140–207: `NameMaxLength = 150`, `INVALID_CATEGORY`, self-parent guard 193–196, `SetActive`).
2. `src/CustomerSupportCrm.Domain/Tickets/Ticket.cs` — `RecordMessage` (317–348): agent public reply sets `LastAgentMessageAt`, `FirstRespondedAt ??=`, New → Open; customer message sets `LastCustomerMessageAt`, reopens PendingCustomer/Resolved; customer on Closed → `TICKET_CLOSED`; raises `TicketMessageAddedDomainEvent`. `SubmitFeedback` (378–394).
3. `src/CustomerSupportCrm.Domain/Attachments/Attachment.cs` — `AttachmentOwnerTypes.Ticket` (9), `ParentId` / `IsPublic` (43, 54), `AttachTo` (95–99, added in `57e52f8`).
4. `src/CustomerSupportCrm.Application/Features/Attachments/AttachmentService.cs` — `StoreAsync` (24–45, `FileUploadRules.ValidateAsync` with `MaxAttachmentBytes` + `Documents`), `GetAsync` (47), `OpenAsync` (53), `DeleteAsync` (62), `ListAsync` (70), `ToResponse` (92), `ToDownload` (96); `NotFoundCode = "ATTACHMENT_NOT_FOUND"` (21). `Common/Files/FileUploadRules.cs` line 16: 20 MB.
5. `src/CustomerSupportCrm.Application/Features/Tickets/Common/TicketQueries.cs` — `GetMessagesAsync` (177–221, `publicOnly` hides internal notes and non-public files), `DownloadPath` (244), `TicketLink` (242).
6. `src/CustomerSupportCrm.Application/Features/Notifications/NotificationSlices.cs` — `NotificationSender.Notify` / `NotifyMany(except:)` (24–45); types in `Domain/Notifications/Notification.cs` lines 67–72.

---

## Backend Tasks

### 1 — Contracts

File: `src/CustomerSupportCrm.Contracts/Tickets/TicketContracts.cs` — `AddTicketMessageRequest(Body, IsInternal, MentionedUserIds?, AttachmentIds?)` (32), `TicketMessageResponse` (101–111, with `Attachments`), `TicketCategoryRequest(Name, NameAr, ParentId, DefaultDepartmentId, DefaultPriority, SortOrder, IsActive = true)` (115), `TicketCategoryResponse` (117–125). `AttachmentResponse` is the shared contract in `Contracts/Common`.

### 2 — Message writer (`TicketMessageSlices.cs` lines 22–62)

`TicketMessageWriter` (scoped, `Application/DependencyInjection.cs` line 45) — the single entry point for every author (agent, portal, inbound channels, chat): `TicketMessage.Create` → `db.TicketMessages.Add` → links pre-uploaded files of this ticket that have no parent (`ParentId == null`) via `AttachTo(message.Id, isPublic: !isInternal)` (47–57) → `ticket.RecordMessage(...)`.

### 3 — Add message (lines 64–109)

- `AddTicketMessageCommand` (64–65); validator (67–75): body 1–50 000, mentions ≤ 20, attachments ≤ 10.
- `AddTicketMessageHandler` (77–109): `LoadAsync` (scope), filters mentions to active users (89–90), writer with `MessageAuthorType.Agent`, `ticket.Channel`; save; returns the projected message.

### 4 — Attachments (lines 111–162)

`UploadTicketAttachmentHandler` (114–124, stored non-public, no parent), `ListTicketAttachmentsHandler` (128–136), `DownloadTicketAttachmentHandler` (140–149), `DeleteTicketAttachmentHandler` (153–162). All call `TicketQueries.EnsureAccessibleAsync` first.

### 5 — Endpoints (lines 164–205)

`TicketMessageEndpoints` on `/tickets/{ticketId:guid}`: `POST /messages` (`tickets.update`), `GET /attachments` (`tickets.view`), `POST /attachments` (`IFormFile file`, `.DisableAntiforgery()`, `tickets.update`), `GET /attachments/{attachmentId}` (`tickets.view`, file result), `DELETE /attachments/{attachmentId}` (`tickets.update`). `GET /tickets/{id}/messages` is in `TicketQuerySlices.cs` lines 187–196 / 245–248.

### 6 — Categories (`TicketCategorySlices.cs`)

- `ListTicketCategoriesQuery(IncludeInactive)` + handler (19–31), ordered by `SortOrder`, `Name`.
- `TicketCategoryMapping.ToResponse` (33–37).
- `SaveTicketCategoryCommand(Guid? CategoryId, TicketCategoryRequest)` (39); validator (41–50, `DefaultPriority` enum name or `INVALID_VALUE`); handler (52–81): unknown parent → 404 `CATEGORY_NOT_FOUND`; create or update; `SetActive`; audit `ticket_categories.created|updated`.
- `TicketCategoryEndpoints` (83–106): `GET /ticket-categories` (no permission), `POST` (201) and `PUT /{id}` with `tickets.categories_manage`.
- Seed: `Infrastructure/Persistence/Seed/DatabaseInitializer.cs` — `DefaultCategories` (31–37: General inquiry, Technical issue, Billing, Complaint, with Arabic names) added when the table is empty (89–92).

### 7 — Conversation / SLA / feedback event handlers (`TicketEventHandlers.cs`)

- `TicketMessageAddedHandler` (142–175): customer message → `ticket.replied` to the assignee; mentions → `ticket.mentioned` (except the author); public messages → customer timeline `TicketMessage`.
- `TicketSlaEventHandlers` (177–204): history `sla_warning` / `sla_breached` + `sla.warning` / `sla.breached` notification to the assignee (events come from SL-01).
- `TicketFeedbackHandler` (206–219): history `feedback` + customer timeline.

### 8 — Persistence

`TicketConfiguration.cs` — `TicketMessageConfiguration` (48–64: `mentioned_user_ids uuid[]`, index (TicketId, CreatedAt), index ExternalMessageId, cascade), `TicketCategoryConfiguration` (81–94: parent FK Restrict, default department SetNull). Tables `ticket_categories` (migration line 486) and `ticket_messages` (917).

### 9 — Localization

`Messages.ar.resx` lines 201–219: `TICKET_CLOSED`, `INVALID_TICKET`, `INVALID_STATUS_TRANSITION`, `INVALID_MESSAGE`, `INVALID_CATEGORY`, `CATEGORY_NOT_FOUND`, `CATEGORY_IN_USE`; `ATTACHMENT_NOT_FOUND` (183). English (`Messages.resx`) only has `CATEGORY_IN_USE` (171) of these; the rest fall back to the exception message.

---

## Edge Cases & Failure Modes

- **Attachment id of another ticket or already linked** — silently ignored by the writer (filter at 51).
- **Files with an internal note** — `IsPublic = false`; the portal (`publicOnly`) never sees them.
- **Unlinked uploads** — appear in `GET /attachments` but in no message; nothing cleans them up.
- **Mention of an inactive/unknown user** — dropped before saving.
- **Reply on a closed ticket by an agent** — allowed by `RecordMessage` (only customers are blocked); no status change.
- **Category cycles** — only self-parenting is rejected; A → B → A is not detected.
- **Deactivated category** — hidden from the default list; `TicketFactory` rejects it on create (404).
- **File too large / disallowed type** — rejected by `FileUploadRules` (validation error).

---

## Test Plan

Test projects are **out of scope** (`tests/` must not be modified). There are no tests in the repo at `2956767` covering ticket messages, ticket attachments or categories.

---

## Verification Steps

1. **Backend builds:** `dotnet build` in `customer-support-crm-api/` — 0 warnings, 0 errors.
2. **Upload:** `curl -X POST …/api/v1/tickets/{id}/attachments -H "Authorization: Bearer $TOKEN" -F "file=@screenshot.png"` → `isPublic: false`, `downloadUrl` `/api/v1/tickets/{id}/attachments/{aid}`.
3. **Reply:** `curl -X POST …/tickets/{id}/messages -d '{"body":"Please try again","isInternal":false,"attachmentIds":["<aid>"]}'` → message with one public attachment; `GET /tickets/{id}` shows `sla.firstRespondedAt` and status `Open`.
4. **Internal note:** `{"body":"Check logs @lead","isInternal":true,"mentionedUserIds":["<lead id>"]}` → `isInternal: true`; the lead gets a `ticket.mentioned` notification (`GET /api/v1/notifications`).
5. **Conversation:** `GET …/tickets/{id}/messages` → both messages in order with author names.
6. **Categories:** `POST /api/v1/ticket-categories {"name":"Refunds","parentId":"<Billing id>","defaultPriority":"High","sortOrder":5}` → 201; `{"defaultPriority":"Critical"}` → 400 `INVALID_VALUE`; Agent token → 403.
7. **Localization:** a failing call with `Accept-Language: ar` → Arabic message.

---

## Done Criteria

- [x] Public reply vs internal note via `isInternal`; internal notes hidden from customer-facing projections.
- [x] Ticket attachments use the shared `AttachmentService` / `FileUploadRules`; files link to messages and inherit visibility.
- [x] Hierarchical categories with routing defaults; list/create/update endpoints.
- [ ] Category delete — not built (`CATEGORY_IN_USE` unused).
- [ ] Configurable priority levels / priority CRUD — not built; fixed enum.
- [ ] History rows for messages — not built.
- [x] Event handlers for messages, SLA warning/breach and feedback write history/timeline/notifications.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to the SLA & Automation feature (Story 28).**
