# 02 — Ticket Management

> **Source:** AZM Squad Customer Support CRM — Core Features §2
> **Implementation phase:** Phase 5 — Tickets
> **Build priority:** 5–7 (Tickets → Assignment/Status/History → Notes & Attachments)

## Summary

The core of the CRM: capture every customer request as a ticket, classify it, route it to the right agent, and track it through a controlled lifecycle until it is resolved.

## Scope

| Sub-feature | Description |
|-------------|-------------|
| Create and track tickets | Create tickets (agent, portal, channel), list/search/filter, view details. |
| Categories and priorities | Configurable categories (hierarchical) and priority levels. |
| Assign tickets to agents | Manual assignment/reassignment to agent, team or department. |
| Status and escalation | Explicit status lifecycle with domain-controlled transitions; manual escalation. |
| Ticket history | Immutable history of every change (status, assignee, priority, category, messages). |

## User stories

- As an **agent**, I can create a ticket linked to a customer with category, priority and description.
- As an **agent**, I can reply to the customer and add internal notes on a ticket.
- As a **supervisor**, I can assign or reassign tickets to agents in my department.
- As an **agent**, I can move a ticket through valid statuses only (e.g. cannot close an unresolved ticket).
- As an **agent**, I can escalate a ticket with a reason.
- As anyone with access, I can see a full history of what happened on the ticket.

## Acceptance criteria

- [ ] Unique, human-readable `TicketNumber` per organization.
- [ ] Status lifecycle: `New → Open → PendingCustomer / PendingInternal → Resolved → Closed`, plus `Escalated`.
- [ ] Invalid transitions rejected by the domain; `Closed → Open` only via a dedicated **Reopen** action.
- [ ] Every change writes a `TicketHistory` entry (who, what, when, old → new).
- [ ] Messages distinguish **public reply** vs **internal note**.
- [ ] Attachments on tickets/messages follow the shared attachment rules.
- [ ] List endpoint supports paging, filtering (status, priority, category, assignee, branch, department, date) and sorting.
- [ ] Domain events raised: `TicketCreated`, `TicketAssigned`, `TicketStatusChanged`, `TicketEscalated`, `TicketResolved`.

## Domain model

```text
Ticket (aggregate)
 ├── TicketMessage
 ├── TicketAttachment
 ├── TicketAssignment
 └── TicketHistory
TicketCategory
TicketPriority
```

## Backend slices

```text
Features/Tickets/
├── CreateTicket
├── UpdateTicket
├── GetTicketById
├── ListTickets
├── AssignTicket
├── ChangeTicketStatus / ResolveTicket / CloseTicket / ReopenTicket
├── EscalateTicket
├── Messages/ (AddReply, AddInternalNote, List)
├── Attachments/
├── GetTicketHistory
├── Categories/ (CRUD)
└── Priorities/ (CRUD)
```

## Frontend

- `features/tickets`: ticket list (filters, saved views), ticket details (conversation, side panel with customer/SLA/assignment), create ticket form, category/priority admin screens.

## Dependencies

- Customer Management (01), Security & Administration (10), Platform (12).
- Feeds: SLA & Automation (05), Agent Dashboard (04), AI Features (07), Reports (09), Communication Channels (03).

## Permissions

`Tickets.View`, `Tickets.Create`, `Tickets.Update`, `Tickets.Assign`, `Tickets.Escalate`, `Tickets.Close`, `Tickets.Reopen`, `Tickets.Categories.Manage`
