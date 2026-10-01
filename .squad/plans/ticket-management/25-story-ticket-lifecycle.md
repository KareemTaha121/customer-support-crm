# Story 25 — Ticket lifecycle: create, update, list, assign, status, escalate, history (Story: TK-01)

> As-built plan: written after implementation. The `Ticket` aggregate, enums, events and records were committed in `customer-support-crm-api` `0fd694e` (feature 01); the slices, contracts and `Ticket.AllowedTransitions` in `57e52f8`; the `tickets` schema in migration `20260930104602_AddSupportOperations` (`678ea67`). `e921626` changed `TicketFactory` (customer lookup). Paths and line numbers refer to `develop` HEAD `2956767`.

## Prerequisites

- Security & Administration stories 01–07 completed ([../security-and-administration/00-overview.md](../security-and-administration/00-overview.md)) — `ICurrentUser` (incl. `HasPermission`), permission policies, `IAuditTrail`, `ApiResults`, `PagedResult`/`PaginationMeta`, `CommonRules`.
- Platform phase 3 (`65c74a3`) — `AccessScope` / `IAccessScopeProvider`, `IScopedEntity`, `ISequenceGenerator`, `NotificationSender`, domain-event dispatch in `ApplicationDbContext`.
- Customer Management (`0fd694e`) — `Customer`, `CustomerQueries.EnsureAccessibleAsync`, `OrganizationUnits.EnsureValidAsync`, `CustomerTimeline`.
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

Staff manage the ticket lifecycle through `/api/v1/tickets`:

1. Create a ticket for a customer (agent or logged phone call) with category, priority, branch/department, tags and an optional assignee.
2. Get one ticket (with `allowedStatuses` and the SLA block) and list tickets with rich filters, search and sorting.
3. Edit subject/description/category/priority/tags.
4. Assign / unassign (self vs others) and transfer to another branch/department.
5. Move through valid statuses only, reopen resolved/closed tickets, escalate with a reason.
6. Soft delete; read the immutable history.

**Deviations from the intake (the code is authoritative):**

| Intake (feature spec 02) | As built |
|---|---|
| `TicketNumber` unique per organization | Single-tenant: one global Postgres sequence `ticket_numbers`, `T-{n:D6}`, unique index on `tickets.number` (`TicketConfiguration.cs` line 20) |
| Separate `ResolveTicket` / `CloseTicket` / `ReopenTicket` slices | One `POST /tickets/{id}/status` with `status` = target status or `"Reopen"` |
| Permissions `Tickets.Close`, `Tickets.Reopen` | Not in the catalog; status, close and reopen all need `tickets.update`. `tickets.delete` added |
| `AssignTicket` needs `Tickets.Assign` | Endpoint needs `tickets.update`; assigning **others** or unassigning additionally needs `tickets.assign` (`ASSIGN_FORBIDDEN`). Transfer needs `tickets.assign` |
| Assign to agent, **team** or department | Agent only (`AssignedAgentId`); department via transfer; no teams |
| `TicketAssignment` entity | No entity; `TicketAssignment` is a static helper class; assignment changes are history rows |
| `TicketResolved` domain event | Not raised; resolution is `TicketStatusChangedDomainEvent` with `Status = Resolved` |
| Closed → Open only via Reopen | Same, and Resolved → Open also only via Reopen (`Transitions[Resolved] = [Closed]`) |
| History for every change | History rows for created, status, assignee, priority, category, department, escalated, SLA, feedback; **no row for subject/description/tag edits or messages** |
| Invalid transition rejected | `DomainException(INVALID_STATUS_TRANSITION)` → **422** |

**Not in scope:** messages, attachments, categories (Story 26); SLA due dates, auto-assignment and escalation rules (SL-01, [../sla-and-automation/28-story-sla-policies-and-automation-engine.md](../sla-and-automation/28-story-sla-policies-and-automation-engine.md)). **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Domain/Tickets/Ticket.cs` — lines 11–462. Constants and error codes (13–21), `Transitions` table (23–32), properties incl. SLA clocks and breach flags (51–125), `AllowedTransitions` (130), `FormatNumber` (132), `Create` (134–175), `Edit` (177–197), `AssignTo` (199–215, New → Open on first assignment), `ChangePriority` (217–228), `Categorize` (230–241), `TransferTo` (243–260), `ChangeStatus` (263–281), `Reopen` (283–293), `Escalate` (295–311), `ChangeStatusInternal` sets `ResolvedAt`/`ClosedAt` (436–453), `EnsureNotClosed` (455–461).
2. `src/CustomerSupportCrm.Domain/Tickets/TicketEnums.cs` — lines 3–51: `TicketStatus`, `TicketPriority` (Low/Medium/High/Urgent), `TicketChannel` (9 channels), `MessageAuthorType`, `SlaTarget`, `IsActive()` (50).
3. `src/CustomerSupportCrm.Domain/Tickets/TicketDomainEvents.cs` — lines 6–26 (11 events).
4. `src/CustomerSupportCrm.Domain/Tickets/TicketRecords.cs` — `TicketHistory` (85–123) and `TicketHistoryActions` (125–137).
5. `src/CustomerSupportCrm.Application/Features/Tickets/Common/TicketQueries.cs` — `TicketErrors` (14–21), `LoadAsync` (28–33), `FindTrackedAsync` (36–38), `EnsureAccessibleAsync` (40–46), `SlaState` (48–53), `ProjectListAsync` (55–115), `GetResponseAsync` (117–174, adds `"Reopen"` to `allowedStatuses` for Resolved/Closed at 141–144), `CanHandleAsync` (227–233), `EligibleAgents` (236–240).
6. `src/CustomerSupportCrm.Application/Common/Authorization/AccessScope.cs` — `WhereInScope` (75–88), `EnsureAccess` → 404 (92–99), `EnsureCanAssign` → 403 `OUT_OF_SCOPE` (102–109).
7. `src/CustomerSupportCrm.Infrastructure/Persistence/ApplicationDbContext.cs` — `DispatchDomainEventsAsync` (101–121): events are published during `SaveChangesAsync` (line 47), up to `MaxDispatchRounds = 10` (23), so handlers add rows to the same unit of work.
8. `src/CustomerSupportCrm.Application/Abstractions/Persistence/ISequenceGenerator.cs` — `NextValueAsync` (6), `Sequences.TicketNumbers` (12); sequence declared in `CustomerConfiguration.cs` line 111.
9. `src/CustomerSupportCrm.Domain/Roles/Permissions.cs` — lines 9–15 ticket permission codes; `AgentDefaults` (65–70) has view/create/update; `ManagerDefaults` (73–82) adds assign/escalate/delete/categories.

---

## Backend Tasks

### 1 — Contracts

File: `src/CustomerSupportCrm.Contracts/Tickets/TicketContracts.cs` (lines 1–125): `CreateTicketRequest` (8–18), `UpdateTicketRequest` (20), `AssignTicketRequest(Guid? AgentId)` (23, null unassigns), `ChangeTicketStatusRequest(string Status)` (25), `EscalateTicketRequest(string Reason)` (27), `TransferTicketRequest(Guid BranchId, Guid? DepartmentId)` (29), `TicketListItemResponse` (35–57, `SlaState` none/ok/warning/breached), `TicketSlaResponse` (59–68), `TicketCustomerResponse` (70), `TicketResponse` (72–99, incl. `AllowedStatuses`), `TicketHistoryResponse` (113).

### 2 — Create (`TicketCommandSlices.cs` lines 24–163)

- `CreateTicketCommand` (26–36); `CreateTicketValidator` (38–49): customer required, subject ≤ 300, description ≤ 20 000, `Priority` enum name (case-insensitive), `Channel` null/`Agent`/`Phone` only (`INVALID_VALUE`), tags ≤ 20.
- `CreateTicketHandler` (51–86): `CustomerQueries.EnsureAccessibleAsync` → `TicketFactory.CreateAsync(NewTicket, scope)` → optional `TicketAssignment.AssignAsync` → save → `GetResponseAsync(AccessScope.Everything)`.
- `TicketFactory` (104–163), registered scoped in `Application/DependencyInjection.cs` line 44 and reused by portal, web form, inbound channels and chat: customer from `db.Customers.Local` first, then DB (112–117, added in `e921626`); active category required (125–126); category `DefaultDepartmentId` sets department + branch when none given (128–132); customer's department as fallback (135–138); `scope?.EnsureCanAssign` (140; null scope for customer/system channels); `OrganizationUnits.EnsureValidAsync` (141); number from sequence (143); `Ticket.Create` (144–156, reply address defaults to customer email).

### 3 — Update (lines 165–200)

`UpdateTicketCommand` / validator (same limits) / handler: `LoadAsync` (scope → 404), category existence check (188–191), then `Edit`, `Categorize`, `ChangePriority` — the latter two raise events only on change.

### 4 — Assign / transfer (lines 202–260)

- `TicketAssignment.AssignAsync` (207–221): self = `agentId == currentUser.UserId`; non-self without `tickets.assign` → `ForbiddenException(ASSIGN_FORBIDDEN)`; `CanHandleAsync` false → `ConflictException(AGENT_NOT_ELIGIBLE)`; then `ticket.AssignTo`.
- `AssignTicketHandler` (226–236).
- `TransferTicketHandler` (241–260): validates unit, `TransferTo`, clears the assignee if no longer eligible (250–253), returns with `AccessScope.Everything` because the caller may have moved it out of scope.

### 5 — Status / escalate / delete (lines 262–325)

- `ChangeTicketStatusValidator` (267–271): `"Reopen"` or a `TicketStatus` name, else `INVALID_VALUE`.
- `ChangeTicketStatusHandler` (273–292): `Reopen(now)` or `ChangeStatus(target, now)`. Requesting `Escalated` → 422 `INVALID_STATUS_TRANSITION` ("Use escalation…").
- `EscalateTicketValidator` reason 1–500 (296–299); `EscalateTicketHandler` (301–311) → `ticket.Escalate(reason, now, automatic: false)`: increments `EscalationLevel`, sets `EscalatedAt`, moves to `Escalated`; Resolved → 422 (reopen first).
- `DeleteTicketHandler` (315–325): `db.Tickets.Remove` (turned into a soft delete by `Infrastructure/Persistence/Interceptors/AuditableEntityInterceptor.cs` lines 38–41), `audit.Record("tickets.deleted", "Ticket", id, oldValues: { Number, Subject, CustomerId })`.

### 6 — Endpoints (lines 327–385)

`TicketCommandEndpoints` maps on `/tickets`: `POST /` (`tickets.create`, 201 + `Location`), `PUT /{id}` (`tickets.update`), `POST /{id}/assign` (`tickets.update`), `POST /{id}/transfer` (`tickets.assign`), `POST /{id}/status` (`tickets.update`), `POST /{id}/escalate` (`tickets.escalate`), `DELETE /{id}` (`tickets.delete`).

### 7 — Queries (`TicketQuerySlices.cs`)

- `ListTicketsQuery` (26–43) + `ListTicketsValidator` (45–60).
- `ListTicketsHandler` (62–177): `WhereInScope`; search on number (exact, upper), subject (contains) or customer name/number inside `#pragma warning disable CA1304, CA1311, CA1862` (70–79); status `active` or csv (81–89); priority csv; channel; assignee `me`/`unassigned`/id (103–116); customer/category/branch/department; `sla` filter on breach/warn flags (138–145); `escalated` (`EscalationLevel > 0`); created range; sorting with priority mapped to 0–3 (162–171) then `ThenBy(Id)`; `PaginationMeta.Create`.
- `GetTicketHandler` (181–185), `GetTicketHistoryHandler` (200–223, actor name from user or customer).
- `TicketQueryEndpoints` (225–255): `GET /tickets`, `/tickets/{id}`, `/tickets/{id}/messages`, `/tickets/{id}/history` — all `tickets.view`.

### 8 — History & lifecycle event handlers (`TicketEventHandlers.cs`)

`TicketHistoryRecorder` (17–33, scoped, DI line 46) stamps acting user or customer and truncates to 500. Lifecycle handlers: `TicketCreatedHandler` (35–45, history `created` + customer timeline), `TicketAssignedHandler` (47–68, history `assignee` with names, `ticket.assigned` notification unless self), `TicketStatusChangedHandler` (70–87, history `status`, timeline on resolve/close/reopen), `TicketFieldChangeHandlers` (89–116, `priority` / `category` / `department` with names), `TicketEscalatedHandler` (118–140, history `escalated` "Level n: reason", notifies `tickets.assign` holders in scope + assignee except the actor).

### 9 — Persistence

`src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/TicketConfiguration.cs` — `TicketConfiguration` (11–46): enums as strings, `tags text[]`, indexes on (Status, Priority), (AssignedAgentId, Status), (BranchId, DepartmentId, Status), (CustomerId, CreatedAt), CreatedAt, ResolutionDueAt; FKs Restrict / SetNull; `xmin` concurrency. `TicketHistoryConfiguration` (66–79). DbSets in `IApplicationDbContext.Tickets.cs` lines 9–15. Tables created in `20260930104602_AddSupportOperations.cs` (`tickets` line 692, `ticket_history` line 893).

### 10 — Localization

`Messages.resx` lines 168–180: `TICKET_NOT_FOUND`, `CATEGORY_IN_USE`, `ASSIGN_FORBIDDEN`, `AGENT_NOT_ELIGIBLE`, `FEEDBACK_NOT_ALLOWED`. `Messages.ar.resx` lines 198–228 also has `TICKET_CLOSED`, `INVALID_TICKET`, `INVALID_STATUS_TRANSITION`, `CATEGORY_NOT_FOUND` (added in `0936711`); in English these fall back to the exception message (`ErrorResponseWriter.Localize`, `Api/Middleware/ErrorResponseWriter.cs` lines 37–42).

---

## Edge Cases & Failure Modes

- **Out-of-scope ticket** — 404 `TICKET_NOT_FOUND`, never 403 (`AccessScope.EnsureAccess`).
- **Create in a unit the caller does not own** — 403 `OUT_OF_SCOPE`; customers/system channels pass `scope: null`.
- **Inactive or unknown category on create** — 404 `CATEGORY_NOT_FOUND`; update only checks existence (an inactive category is accepted).
- **Same value** — `AssignTo`, `ChangePriority`, `Categorize`, `TransferTo`, `ChangeStatus` are no-ops without events or history.
- **Closed ticket** — every mutator except `Reopen` throws `TICKET_CLOSED` (422).
- **Escalate twice** — level increments again; status stays `Escalated`.
- **Reopen** — clears `ResolvedAt`/`ClosedAt`; breach flags and due dates are not recomputed.
- **Concurrent edits** — `xmin` → 409 `CONFLICT`.
- **Auto close** — `AutoCloseResolvedTicketsCommand` (`Features/Settings/SettingsSlices.cs` 163–185, hourly, `678ea67`) closes tickets resolved longer than the `tickets.auto_close_resolved_days` setting (default 7, `Domain/Integrations/Integrations.cs` lines 304, 313).

---

## Test Plan

Test projects are **out of scope** (`tests/` must not be modified). No tests exist in the repo for tickets: `tests/` has no ticket domain, application or integration tests at `2956767`. Only `tests/CustomerSupportCrm.IntegrationTests/RoleManagementTests.cs` line 120 checks that `tickets.assign` is in the permission catalog.

---

## Verification Steps

1. **Backend builds:** in `customer-support-crm-api/` run `dotnet build` — 0 warnings, 0 errors.
2. **Run:** migrate + seed (bootstrap admin, default branch/department, categories, Standard SLA), sign in, export `TOKEN`.
3. **Create:** `curl -i -X POST https://localhost:<port>/api/v1/tickets -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"customerId":"<id>","subject":"Cannot log in","priority":"High"}'` → 201, `number` `T-000001`, `status` `New`, `allowedStatuses` `[Open, PendingCustomer, PendingInternal, Resolved]`, `sla.firstResponseDueAt` set.
4. **List:** `curl "…/api/v1/tickets?status=active&priority=High,Urgent&assignee=unassigned&sortBy=priority&pageSize=10"` → `meta` present; `status=Foo` → 400 `INVALID_VALUE`.
5. **Assign:** `POST …/tickets/{id}/assign {"agentId":"<own id>"}` as Agent → 200, status `Open`; another agent's id as Agent → 403 `ASSIGN_FORBIDDEN`.
6. **Status:** `POST …/status {"status":"Closed"}` on an Open ticket → 422 `INVALID_STATUS_TRANSITION`; `Resolved` then `Closed` → 200; `PUT` on the closed ticket → 422 `TICKET_CLOSED`; `{"status":"Reopen"}` → `Open`.
7. **Escalate:** `POST …/escalate {"reason":"VIP"}` → `escalationLevel: 1`, `status: Escalated`.
8. **History:** `GET …/tickets/{id}/history` → `created`, `assignee`, `status`, `escalated` rows with actor names.
9. **Permissions:** Agent token on `POST …/transfer` or `DELETE …/tickets/{id}` → 403.

---

## Done Criteria

- [x] Unique human-readable `T-000123` number (global sequence + unique index; single tenant).
- [x] Status lifecycle with `Escalated`; transitions enforced by `Ticket.Transitions`; Reopen is the only way back from Resolved/Closed.
- [x] Create, update, get, list (paging, filters incl. status/priority/category/assignee/branch/department/date/SLA, sorting), assign, transfer, status, escalate, soft delete endpoints with their permissions.
- [ ] Every change writes history — not for subject/description/tag edits or messages (see deviations).
- [ ] `TicketResolved` event — not built; resolution is a `TicketStatusChanged` event.
- [ ] Separate `tickets.close` / `tickets.reopen` permissions — not built; `tickets.update` covers both.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to Story 26.**
