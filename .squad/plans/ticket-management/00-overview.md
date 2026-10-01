# Ticket Management

Feature spec: [../../features/02-ticket-management.md](../../features/02-ticket-management.md).
Repo: `customer-support-crm-api`. Out of scope for every story: Docker, deployment, CI/CD, and files under `tests/`.
Matching frontend plan: [../frontend/11-story-tickets-ui.md](../frontend/11-story-tickets-ui.md) (FE-04).

| NN | Tracker id | Plan file | Title | Depends on | Status |
|----|-----------|-----------|-------|-----------|--------|
| 25 | TK-01 | [25-story-ticket-lifecycle.md](25-story-ticket-lifecycle.md) | Ticket lifecycle: create, update, list, assign, status, escalate, history | Phase 2, Phase 3, feature 01 | Done |
| 26 | TK-02 | [26-story-ticket-conversation-and-categories.md](26-story-ticket-conversation-and-categories.md) | Ticket conversation, attachments, categories and event handlers | 25 | Done |
| 43 | BUG-07 | [43-story-ticket-category-cycle-guard.md](43-story-ticket-category-cycle-guard.md) | Prevent cycles in ticket category parents | 26 | Done (`cedce62`) |
| 51 | BUG-15 | [51-story-localized-category-names.md](51-story-localized-category-names.md) | Arabic ticket category names everywhere a category is shown (QA M2) | 26, 34 | To do |

Both plans are **as-built** plans written after implementation, from the real code. Their paths and line numbers refer to `develop` HEAD `2956767`.

Commits:

- `0fd694e` (feat: add customer management) — the `Ticket` aggregate, `TicketEnums`, `TicketDomainEvents`, `TicketMessage` / `TicketHistory` / `TicketCategory` records (committed early because customers link to tickets).
- `57e52f8` (feat: add ticket management) — `Features/Tickets/*` slices, `TicketContracts`, event handlers, `Ticket.AllowedTransitions`, `Attachment.AttachTo`. It also added the SLA/automation **domain** (`Domain/Sla/*`, `SlaConfiguration.cs`, `IApplicationDbContext.Automation.cs`), which [../sla-and-automation/00-overview.md](../sla-and-automation/00-overview.md) covers.
- `b81ba28` — default ticket categories seeded; SLA due dates and assignment rules run on `TicketCreated`.
- `e921626` — `TicketFactory` also finds customers created earlier in the same unit of work (inbound channels).
- `678ea67` — migration `20260930104602_AddSupportOperations` creates the ticket tables; hourly `AutoCloseResolvedTicketsCommand` (Settings).
- `0936711` — Arabic messages for the ticket error codes.
- `889dfcf` — `GET /kb/suggestions?ticketId=` (knowledge base feature) matches any term of the ticket subject.

Known gaps and drift (spec vs as built):

- **No `TicketResolved` domain event**; resolution is a `TicketStatusChanged` event.
- **No `tickets.close` / `tickets.reopen` permissions**; `POST /tickets/{id}/status` (incl. `"Reopen"`) needs `tickets.update`. A `tickets.delete` permission and soft delete were added.
- **Assignment to agent only** (no team); department via `POST /tickets/{id}/transfer`.
- **History is not complete**: no rows for subject/description/tag edits or messages.
- **Priorities are a fixed enum** (Low/Medium/High/Urgent); no priority CRUD. **Categories have no delete** (`CATEGORY_IN_USE` declared but unused); ~~category cycles are not detected~~ (fixed by story 43, `cedce62`; [intake](../../stories/ticket-management/ticket-category-cycle-guard/intake.md)).
- **No dedicated attachment entity**; the shared `Attachment` table is used. Unlinked uploads are never cleaned up.
- **Ticket number is global**, not per organization (single-tenant).
- **Reopen does not recompute SLA** due dates or clear breach flags.
- **English resx** lacks several domain codes (`TICKET_CLOSED`, `INVALID_STATUS_TRANSITION`, `CATEGORY_NOT_FOUND`, …); English responses use the exception message. → BUG-09 ([intake](../../stories/platform/missing-error-messages/intake.md))
- **No tests** for tickets in `tests/`.
