# Story 31 — Portal tickets, history and satisfaction feedback (Story: CP-02)

> As-built plan: written after implementation in `customer-support-crm-api` commit `00d3f35` (feat: add customer portal). `PortalTicketSlices.cs` and `PortalContracts.cs` have not changed since, so their line numbers are the same at `00d3f35` and `develop` HEAD `2956767`; line numbers for other files refer to `2956767`.

## Prerequisites

- Story 30 completed: [30-story-portal-accounts-and-authentication.md](30-story-portal-accounts-and-authentication.md) — customer tokens, `ICurrentCustomer`, `/api/v1/portal` group with the customer policy.
- Ticket Management (02): `Ticket` aggregate (status transitions, `RecordMessage`, `SubmitFeedback`), `TicketFactory`, `TicketMessageWriter`, `TicketQueries`, `AttachmentService`.
- Communication Channels (03, `e921626`): resolution notice to the customer (`CustomerMessenger.QueueResolvedAsync`).
- Knowledge Base story 29: [../knowledge-base/29-story-knowledge-base-articles-and-search.md](../knowledge-base/29-story-knowledge-base-articles-and-search.md) — public FAQ/article endpoints the portal uses for "Access FAQs".
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

1. A signed-in customer submits a ticket (subject, message, optional category) and receives the ticket number.
2. They list their own tickets (open / closed / all), open one, read the public conversation and reply with attachments.
3. They see their activity history.
4. Once a ticket is resolved they rate it (1–5 + comment) and can confirm it is solved / close it.
5. They browse and search public FAQs through the anonymous KB endpoints (story 29).

Every ticket query is filtered by the token's customer id; internal notes and internal attachments are never returned.

**Deviations from the intake (the code is authoritative):**

| Intake / feature spec | As built in `00d3f35` |
|---|---|
| `Features/Portal/Tickets/(Submit, ListMine, GetMine, Reply)`, `Feedback/(Submit)` | One file `Features/CustomerPortal/PortalTicketSlices.cs`; plus upload/download attachment, close, categories and history (the last two are inline endpoint lambdas) |
| `Features/Portal/KnowledgeBase/(Browse, Search)` | **No portal KB slice** — the portal uses `/api/v1/public/kb/*` from story 29 |
| Customer sees own tickets **or their company's, per role** | Own tickets only (`t.CustomerId == customer.CustomerId`); no company/role sharing |
| Only public statuses with customer-friendly labels | `Status` is the raw enum name (`New`, `Open`, `PendingCustomer`, `PendingInternal`, `Escalated`, `Resolved`, `Closed`); mapping to labels is left to the client |
| Assignment details never exposed | `PortalTicketResponse.AgentName` returns the assigned agent's display name (no ids, team, SLA or routing) |
| CSAT survey triggered on `TicketResolved`; **one response per ticket** | Resolution email "Rate our service" is queued on `TicketStatusChanged → Resolved` (Channels); feedback is allowed while `Resolved` or `Closed` and a second submission **overwrites** the first |
| `CustomerFeedback (Rating, Comment, TicketId)` entity | No separate entity — `Ticket.SatisfactionRating`, `SatisfactionComment`, `SatisfactionSubmittedAt` |
| Deflection while typing the subject | No dedicated endpoint; the client calls `GET /api/v1/public/kb/articles?search=…` (all-terms match, see story 29) |

**Not in scope:** branding endpoints (Platform 12), company-wide visibility. **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Application/Features/CustomerPortal/PortalTicketSlices.cs` — `PortalTickets` (22–53: `LoadOwnAsync` 25–30, `DownloadPath` 32, `ToResponseAsync` 34–52), list (55–76), create (78–103), get (105–111), messages (113–124), reply (126–145), upload (147–158), download (160–170), feedback (172–192), close (194–206), `PortalTicketEndpoints : IPortalEndpoint` (208–292).
2. `src/CustomerSupportCrm.Contracts/Portal/PortalContracts.cs` — `PortalCreateTicketRequest` (22), `PortalMessageRequest` (24), `PortalFeedbackRequest` (26), `PortalTicketResponse` (29–42), `PortalTicketListItemResponse` (44), `PortalMessageResponse` (46), `PortalCategoryResponse` (48), `PortalHistoryItemResponse` (50).
3. `src/CustomerSupportCrm.Domain/Tickets/Ticket.cs` — codes `INVALID_STATUS_TRANSITION` / `TICKET_CLOSED` / `FEEDBACK_NOT_ALLOWED` (19–21), `Transitions` (23–32), `ChangeStatus` (263–281), `RecordMessage` (316–345: customer on Closed → `TICKET_CLOSED`; customer reply reopens `PendingCustomer`/`Resolved`), `SubmitFeedback` (378–394, raises `TicketFeedbackSubmittedDomainEvent`).
4. `src/CustomerSupportCrm.Application/Features/Tickets/Common/TicketQueries.cs` — `NotFound()` (25, `TICKET_NOT_FOUND`), `GetMessagesAsync(…, publicOnly, …)` (177–; filters `!m.IsInternal` and `a.IsPublic`).
5. `src/CustomerSupportCrm.Application/Features/Tickets/TicketCommandSlices.cs` — `NewTicket` (88), `TicketFactory.CreateAsync(request, scope, ct)` (104–107); `src/CustomerSupportCrm.Application/Features/Tickets/TicketMessageSlices.cs` — `TicketMessageWriter` (26).
6. `src/CustomerSupportCrm.Application/Features/Attachments/AttachmentService.cs` — `StoreAsync` (24), `GetAsync(…, publicOnly)` (47–49).
7. `src/CustomerSupportCrm.Application/Features/Channels/CustomerMessaging.cs` — `CustomerTemplates.Resolved` (52–55, "Rate our service"), `QueueResolvedAsync` (106–112), resolved handler (217–228, skips chat tickets).
8. `src/CustomerSupportCrm.Application/Features/Tickets/TicketEventHandlers.cs` — `TicketFeedbackHandler` (205–) records ticket history and the customer timeline; `Features/Integrations/IntegrationSlices.cs` line 229 forwards the event to webhooks.
9. `src/CustomerSupportCrm.Domain/Customers/CustomerRecords.cs` — `CustomerActivityTypes` (120–127) used by `/history`.

---

## Backend Tasks

### 1 — Contracts

Add to `src/CustomerSupportCrm.Contracts/Portal/PortalContracts.cs` the ticket records listed above. `PortalTicketResponse(Id, Number, Subject, Description, Status, CategoryName, AgentName, CanReply, CanGiveFeedback, SatisfactionRating, SatisfactionComment, CreatedAt, UpdatedAt)` — no priority, SLA, team, branch, tags or internal data. `PortalMessageResponse` reuses `Contracts.Common.AttachmentResponse`.

### 2 — Ownership helper

`PortalTickets.LoadOwnAsync` loads `db.Tickets` tracked with `t.Id == ticketId && t.CustomerId == customer.CustomerId`, else `TicketQueries.NotFound()` (404 for other customers' tickets too). `ToResponseAsync` computes `CanReply = Status != Closed`, `CanGiveFeedback = Status is Resolved or Closed`.

### 3 — List and get

- `PortalListTicketsQuery(Page = 1, PageSize = 20, Status = null)` (55–76): `closed` = Resolved/Closed, `all`, anything else = active; ordered by `CreatedAt` desc; page ≥ 1, size clamped 1–50 (no validator — clamped in the handler); projects `LastAgentMessageAt` as `LastAgentReplyAt`.
- `PortalGetTicketQuery` (105–111).

### 4 — Create

`PortalCreateTicketCommand(Subject, Message, CategoryId)` + validator (subject ≤ `Ticket.SubjectMaxLength`, message ≤ 10 000) + handler (89–103): category kept only if it exists and is active; requester email from the account; `TicketFactory.CreateAsync(new NewTicket(customerId, subject, message, categoryId, TicketPriority.Medium, TicketChannel.Portal, null, null, [], email), scope: null)`; save → 201 `Location: /api/v1/portal/tickets/{id}`. Ticket number, routing, SLA and the "request received" notice come from the shared ticket pipeline.

### 5 — Messages and attachments

- `PortalGetMessagesQuery` (113–124): `GetMessagesAsync(publicOnly: true)` with portal download URLs.
- `PortalAddMessageCommand(TicketId, Body, AttachmentIds)` + validator (body ≤ 10 000, ≤ 10 attachments) (126–145): `TicketMessageWriter.AddAsync(ticket, MessageAuthorType.Customer, null, customerId, body, isInternal: false, TicketChannel.Portal, …)`.
- `PortalUploadAttachmentCommand` (147–158): `AttachmentService.StoreAsync(Ticket, ticketId, …, isPublic: true, …, customerId)`; endpoint `.DisableAntiforgery()`, multipart `IFormFile`.
- `PortalDownloadAttachmentQuery` (160–170): `GetAsync(publicOnly: true)` then `OpenAsync`; returned with `AttachmentService.ToDownload`.

### 6 — Feedback and close

- `PortalFeedbackCommand(TicketId, Rating, Comment)` + validator (rating 1–5, comment ≤ 2000) (172–192) → `ticket.SubmitFeedback(rating, comment, now)`.
- `PortalCloseTicketCommand` (194–206): `Resolved → Closed`, otherwise `→ Resolved` via `ChangeStatus` (domain transition rules apply).

### 7 — Endpoints

`PortalTicketEndpoints` (208–292) under `/api/v1/portal`: `GET /categories` (active ticket categories ordered by `SortOrder`, `Name`), `GET /history?page` (activity types `ticket.created`, `ticket.status_changed`, `ticket.message`, `ticket.feedback`, `portal.sign_in`, `chat.started`; 25 per page), and the `/tickets` group. Names `PortalListCategories`, `PortalHistory`, `PortalListTickets`, `PortalCreateTicket`, `PortalGetTicket`, `PortalGetMessages`, `PortalAddMessage`, `PortalUploadAttachment`, `PortalDownloadAttachment`, `PortalSubmitFeedback`, `PortalCloseTicket`. No per-endpoint permission: the group's customer policy is the only requirement.

### 8 — Localization and docs

Codes reused from Ticket Management: `TICKET_NOT_FOUND` (en 168 / ar 198), `FEEDBACK_NOT_ALLOWED` (en 180 / ar 228), `ATTACHMENT_NOT_FOUND` (ar 183); `TICKET_CLOSED` / `INVALID_STATUS_TRANSITION` from the ticket domain. `docs/endpoints.md` "Customer portal" (182–194).

---

## Edge Cases & Failure Modes

- **Another customer's ticket id** — 404 `TICKET_NOT_FOUND` on every route (no existence leak).
- **Internal notes / internal files** — excluded by `publicOnly: true` in messages and downloads; an internal attachment id → 404.
- **Inactive or unknown category on create** — silently dropped, ticket still created.
- **Reply to a Closed ticket** — 422 `TICKET_CLOSED`; reply to Resolved reopens it to `Open` (and clears `ResolvedAt`).
- **Feedback before resolution** — 422 `FEEDBACK_NOT_ALLOWED`; rating outside 1–5 → 400 validation error; resubmitting overwrites the earlier rating.
- **Close twice** — first `Resolved`, second `Closed`, third → 422 `INVALID_STATUS_TRANSITION`.
- **Revoked account** — an unexpired token can still use these routes (see story 30).
- **Large uploads** — limited by the attachment rules in `AttachmentService` and the global request-body limit (`2956767`).
- **Concurrent updates on a ticket** — `xmin` token → 409 `CONFLICT`.

---

## Test Plan

Test projects are **out of scope** (`tests/` must not be modified). No tests are added, changed or removed by this story. There are **no existing tests** in `2956767` that cover the portal ticket routes.

---

## Verification Steps

1. **Backend builds:** in `customer-support-crm-api/` run `dotnet build` — 0 warnings, 0 errors.
2. **Sign in** as a portal customer (story 30) and export `PTOKEN`.
3. **Categories:** `curl https://localhost:<port>/api/v1/portal/categories -H "Authorization: Bearer $PTOKEN"` → active categories.
4. **Deflection:** `curl "https://localhost:<port>/api/v1/public/kb/articles?search=password"` → published public articles.
5. **Create:** `POST /api/v1/portal/tickets -d '{"subject":"Cannot reset my password","message":"…"}'` → 201 with `number`, `status: "New"`.
6. **List / get:** `GET /api/v1/portal/tickets` → the ticket; `?status=closed` → empty; `GET /tickets/{id}` with another customer's id → 404.
7. **Conversation:** as staff add an internal note and a public reply; `GET /api/v1/portal/tickets/{id}/messages` → only the public reply. Upload a file with `curl -F file=@a.pdf …/tickets/{id}/attachments`, then `POST …/messages -d '{"body":"see file","attachmentIds":["<id>"]}'` → 200; download → file.
8. **Feedback:** `POST …/feedback -d '{"rating":5}'` while open → 422 `FEEDBACK_NOT_ALLOWED`; `POST …/close` → `Resolved`, `canGiveFeedback: true`; feedback again → 200 with `satisfactionRating: 5`.
9. **Close:** `POST …/close` → `Closed`; reply → 422 `TICKET_CLOSED`.
10. **History:** `GET /api/v1/portal/history` → ticket and sign-in entries, paged.
11. **Isolation:** staff token on `/api/v1/portal/tickets` → 403.

---

## Done Criteria

- [x] Customers submit tickets (number returned), list them (open/closed/all), view details and reply with attachments.
- [x] Only the customer's own tickets are reachable; other ids return 404.
- [ ] Company-level visibility per role — not built.
- [x] Internal notes and internal attachments never returned; no SLA/routing/priority in responses. Assigned agent **name** is exposed (see deviations).
- [ ] Customer-friendly public statuses — raw enum names returned.
- [x] CSAT: rating 1–5 + comment after resolution; resolution email invites a rating. [ ] One response per ticket — not enforced (resubmission overwrites).
- [x] FAQ access through the public KB endpoints (story 29); [ ] no dedicated deflection endpoint.
- [x] Activity history endpoint.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/` by this story.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding.**
