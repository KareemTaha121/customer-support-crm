# Story intake

- Folder: `.squad/stories/ticket-management/ticket-lifecycle/intake.md`

---

## Feature

- **Feature name (display):** Ticket Management
- **Feature slug (folder under `plans/`):** `ticket-management`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `TK-01`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `tickets`

---

## Title

```
Ticket lifecycle: create, update, list, assign, status, escalate, history
```

---

## Description

```
Staff create and work tickets through /api/v1/tickets in customer-support-crm-api.
Slices are grouped per file in Application/Features/Tickets/ (TicketCommandSlices.cs,
TicketQuerySlices.cs, Common/TicketQueries.cs). Contracts in Contracts/Tickets/TicketContracts.cs.
Domain aggregate Domain/Tickets/Ticket.cs owns status transitions and raises domain events.

Endpoints and permissions:
- POST   /tickets                  tickets.create    Create {customerId, subject, description, categoryId,
                                                      priority, channel (Agent|Phone), branchId, departmentId,
                                                      tags[], assignedAgentId}
- GET    /tickets/{id}             tickets.view      Get (with allowedStatuses, SLA block, customer summary)
- GET    /tickets                  tickets.view      List: page, pageSize (max 100), search (number/subject/
                                                      customer), status (csv or "active"), priority (csv),
                                                      channel, assignee (me|unassigned|id), customerId,
                                                      categoryId, branchId, departmentId, sla (breached|warning|
                                                      at_risk), escalated, createdFrom, createdTo,
                                                      sortBy (createdAt|updatedAt|priority|dueAt|number), sortDirection
- PUT    /tickets/{id}             tickets.update    Update {subject, description, categoryId, priority, tags[]}
- POST   /tickets/{id}/assign      tickets.update    Assign {agentId|null}; self only, others/unassign need tickets.assign
- POST   /tickets/{id}/transfer    tickets.assign    Transfer {branchId, departmentId}
- POST   /tickets/{id}/status      tickets.update    ChangeStatus {status: <TicketStatus> | "Reopen"}
- POST   /tickets/{id}/escalate    tickets.escalate  Escalate {reason}
- DELETE /tickets/{id}             tickets.delete    Soft delete (audited)
- GET    /tickets/{id}/history     tickets.view      Ordered history rows with actor name

Rules:
- TicketNumber "T-000123" from the ticket_numbers sequence; unique index on tickets.number.
- Statuses New, Open, PendingCustomer, PendingInternal, Escalated, Resolved, Closed; transition table in
  Ticket.cs; Escalated only via escalate; Resolved/Closed -> Open only via "Reopen";
  Closed tickets are read-only (TICKET_CLOSED, 422).
- Invalid transition -> 422 INVALID_STATUS_TRANSITION.
- Unknown / out-of-scope ticket -> 404 TICKET_NOT_FOUND; unknown category -> 404 CATEGORY_NOT_FOUND.
- Assigning someone else without tickets.assign -> 403 ASSIGN_FORBIDDEN; agent inactive or outside the
  ticket's branch/department -> 409 AGENT_NOT_ELIGIBLE; create/transfer into a unit outside the caller's
  scope -> 403 OUT_OF_SCOPE.
- Domain events: TicketCreated, TicketAssigned, TicketStatusChanged, TicketPriorityChanged,
  TicketCategoryChanged, TicketTransferred, TicketEscalated; handlers write ticket_history rows.
```

---

## Acceptance criteria

```
- [ ] Unique, human-readable `TicketNumber` per organization.
- [ ] Status lifecycle: `New → Open → PendingCustomer / PendingInternal → Resolved → Closed`, plus `Escalated`.
- [ ] Invalid transitions rejected by the domain; `Closed → Open` only via a dedicated **Reopen** action.
- [ ] Every change writes a `TicketHistory` entry (who, what, when, old → new).
- [ ] List endpoint supports paging, filtering (status, priority, category, assignee, branch, department, date) and sorting.
- [ ] Domain events raised: `TicketCreated`, `TicketAssigned`, `TicketStatusChanged`, `TicketEscalated`, `TicketResolved`.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** Customer Management (feature 01), Security & Administration P2-01..P2-07, Platform (phase 3 organization scope).
- **Depends on code areas or other stories:** `Customer`, `AccessScope` / `IAccessScopeProvider`, `ISequenceGenerator`, `OrganizationUnits`, `ICurrentUser.HasPermission`, `IAuditTrail`, domain-event dispatch in `ApplicationDbContext`.

## Extra notes (optional)

- The `Ticket` aggregate, events, enums and records were committed with Customer Management (`0fd694e`); the slices with `57e52f8`.
- SLA due dates and auto-assignment on `TicketCreated` belong to SLA & Automation (SL-01).

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- Messages, attachments, categories (TK-02); SLA engine (SL-01); portal ticket endpoints (feature 08).
