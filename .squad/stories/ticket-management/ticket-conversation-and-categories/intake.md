# Story intake

- Folder: `.squad/stories/ticket-management/ticket-conversation-and-categories/intake.md`

---

## Feature

- **Feature name (display):** Ticket Management
- **Feature slug (folder under `plans/`):** `ticket-management`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `TK-02`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `tickets`

---

## Title

```
Ticket conversation (replies, internal notes, attachments), categories and event handlers
```

---

## Description

```
Agents talk to the customer and to each other on a ticket, attach files, and admins
maintain ticket categories. Code in Application/Features/Tickets/TicketMessageSlices.cs,
TicketCategorySlices.cs and TicketEventHandlers.cs; contracts in Contracts/Tickets.

Endpoints and permissions:
- GET    /tickets/{id}/messages                     tickets.view    Conversation incl. internal notes and files
- POST   /tickets/{id}/messages                     tickets.update  Add {body, isInternal, mentionedUserIds[] (≤20),
                                                                    attachmentIds[] (≤10, pre-uploaded)}
- GET    /tickets/{id}/attachments                  tickets.view    List ticket files
- POST   /tickets/{id}/attachments                  tickets.update  Upload (multipart "file"); unlinked until a message references it
- GET    /tickets/{id}/attachments/{attachmentId}   tickets.view    Download
- DELETE /tickets/{id}/attachments/{attachmentId}   tickets.update  Delete
- GET    /ticket-categories?includeInactive=        authenticated   List (SortOrder, Name)
- POST   /ticket-categories                         tickets.categories_manage  Create {name, nameAr, parentId,
                                                                    defaultDepartmentId, defaultPriority, sortOrder, isActive}
- PUT    /ticket-categories/{id}                    tickets.categories_manage  Update (same body)

Rules:
- Public reply vs internal note via isInternal; only agents can write internal notes (INVALID_MESSAGE).
- An agent public reply sets FirstRespondedAt (first time) and moves New -> Open; a customer message
  reopens PendingCustomer/Resolved tickets; customers cannot write on Closed tickets (TICKET_CLOSED).
- Files use the shared AttachmentService / FileUploadRules (20 MB, document allow-list); files sent with
  an internal note stay non-public.
- Mentions are filtered to active users and notified (ticket.mentioned).
- Category name 1-150 chars, cannot be its own parent; unknown parent/category -> 404 CATEGORY_NOT_FOUND.
- Event handlers: history rows for created/assignee/status/priority/category/department/escalated/
  sla_warning/sla_breached/feedback; customer timeline entries; in-app notifications
  (ticket.assigned, ticket.replied, ticket.mentioned, ticket.escalated, sla.warning, sla.breached).
```

---

## Acceptance criteria

```
- [ ] Messages distinguish **public reply** vs **internal note**.
- [ ] Attachments on tickets/messages follow the shared attachment rules.
- [ ] Configurable categories (hierarchical) and priority levels.
- [ ] Every change (including messages) writes a `TicketHistory` entry.
- [ ] Category / priority CRUD.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** TK-01.
- **Depends on code areas or other stories:** `AttachmentService`, `FileUploadRules`, `NotificationSender`, `CustomerTimeline`, `ICurrentCustomer`.

## Extra notes (optional)

- Delivery of public replies over email/WhatsApp/SMS/chat is done by the channels feature (03) on `TicketMessageAdded`.
- Article suggestions for a ticket (`GET /kb/suggestions?ticketId=`) belong to the knowledge base feature (06).

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- Customer-portal messages and feedback endpoints (feature 08).
