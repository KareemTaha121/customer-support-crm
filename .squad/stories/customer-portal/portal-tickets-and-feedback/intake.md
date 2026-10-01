# Story intake

- Folder: `.squad/stories/customer-portal/portal-tickets-and-feedback/intake.md`

---

## Feature

- **Feature name (display):** Customer Portal
- **Feature slug (folder under `plans/`):** `customer-portal`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `CP-02`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `customer-portal`

---

## Title

```
Portal tickets, history and satisfaction feedback
```

---

## Description

```
Signed-in customers submit and follow their own tickets, read and send public messages with
attachments, see their activity history, rate resolved tickets and close them. Slices in
Application/Features/CustomerPortal/PortalTicketSlices.cs; contracts in Contracts/Portal/PortalContracts.cs.
FAQ access uses the anonymous knowledge base endpoints of KB-01 (/api/v1/public/kb/...).

Customer endpoints (/api/v1/portal, customer policy; all scoped to the token's customer id):
- GET  /categories                                 Active ticket categories {id, name, nameAr}
- GET  /history?page                               Customer activity timeline (portal-safe types), 25/page
- GET  /tickets?page&pageSize&status               Own tickets; status open (default) | closed | all;
                                                   pageSize 1-50 (default 20), newest first
- POST /tickets                                    {subject, message, categoryId?} -> 201 PortalTicketResponse
                                                   (channel Portal, priority Medium, ticket number)
- GET  /tickets/{id}                               PortalTicketResponse (no SLA, routing or notes)
- GET  /tickets/{id}/messages                      Public messages only, with public attachments
- POST /tickets/{id}/messages                      {body, attachmentIds[] (max 10)}
- POST /tickets/{id}/attachments                   multipart file -> AttachmentResponse
- GET  /tickets/{id}/attachments/{attachmentId}    Download (public attachments of own ticket)
- POST /tickets/{id}/feedback                      {rating 1-5, comment <= 2000} -> PortalTicketResponse
- POST /tickets/{id}/close                         Active -> Resolved, Resolved -> Closed

Rules:
- Another customer's ticket or unknown id -> 404 TICKET_NOT_FOUND.
- Unknown or inactive categoryId on create is dropped (ticket created without category).
- Feedback only when Resolved or Closed -> otherwise 422 FEEDBACK_NOT_ALLOWED; raises
  TicketFeedbackSubmittedDomainEvent.
- Customer reply on a Closed ticket -> 422 TICKET_CLOSED; on Resolved/PendingCustomer it reopens to Open.
- Close on a Closed ticket -> 422 INVALID_STATUS_TRANSITION.
- Resolution notice to the customer (asking for a rating) is queued on TicketStatusChanged -> Resolved
  by the Channels feature (non-chat tickets).
- Messages localized (en/ar).
```

---

## Acceptance criteria

```
- [ ] As a customer, I submit a ticket and receive a ticket number.
- [ ] As a customer, I see the status of my tickets and reply to the agent.
- [ ] As a customer, I view my past tickets.
- [ ] As a customer, I search FAQs before submitting a ticket.
- [ ] As a customer, I rate my experience when a ticket is resolved.
- [ ] Customer sees only their own tickets (or their company's, per role).
- [ ] Internal notes, internal articles, assignment details and agent-only data are never exposed.
- [ ] Only public ticket statuses shown (mapped customer-friendly labels).
- [ ] CSAT survey triggered on TicketResolved; one response per ticket.
- [ ] Deflection: suggested FAQs shown while typing the ticket subject.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** CP-01, KB-01, Ticket Management (02), Communication Channels (03).
- **Depends on code areas or other stories:** `TicketFactory`, `TicketMessageWriter`, `TicketQueries.GetMessagesAsync(publicOnly)`, `AttachmentService`, `Ticket.SubmitFeedback` / `ChangeStatus`, `CustomerActivities`, `ICurrentCustomer`.

## Extra notes (optional)

- Feature spec: `.squad/features/08-customer-portal.md` (Tickets, KnowledgeBase, Feedback slices).
- CSAT reporting reads `Ticket.SatisfactionRating` (Reports, feature 09).
- Frontend: `.squad/plans/frontend/17-story-customer-portal-ui.md` (FE-10).

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.
- Implemented in commit `00d3f35`.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- Company-wide ticket visibility, organization branding endpoints (Platform 12).
