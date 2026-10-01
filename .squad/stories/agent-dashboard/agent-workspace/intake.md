# Story intake

- Folder: `.squad/stories/agent-dashboard/agent-workspace/intake.md`

---

## Feature

- **Feature name (display):** Agent Dashboard
- **Feature slug (folder under `plans/`):** `agent-dashboard`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `AD-01`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `agent-dashboard`

---

## Title

```
Agent workspace: dashboard, tasks & reminders, quick replies
```

---

## Description

```
The agent's home screen and personal tools in customer-support-crm-api, all in
Application/Features/Dashboard/AgentWorkspaceSlices.cs. Contracts in
Contracts/Dashboard/DashboardContracts.cs; entities AgentTask and QuickReply in
Domain/Tickets/AgentWorkspace.cs.

Endpoints and permissions:
- GET    /dashboard/agent             tickets.view   Counts + short lists (10; recent customers 8)
- GET    /tasks                       authenticated  status (open|completed|all), assignee (me|userId),
                                                     ticketId, customerId; max 200
- POST   /tasks                       authenticated  {title, notes, assigneeId, ticketId, customerId, dueAt, remindAt}
- PUT    /tasks/{id}                  authenticated  Same body
- POST   /tasks/{id}/complete         authenticated
- POST   /tasks/{id}/reopen           authenticated
- DELETE /tasks/{id}                  authenticated
- GET    /quick-replies               authenticated  search (title, /shortcut prefix, body), language; max 100
- POST   /quick-replies               authenticated  {title, shortcut, body, language (en|ar), categoryId, shared}
- PUT    /quick-replies/{id}          authenticated
- DELETE /quick-replies/{id}          authenticated
- POST   /quick-replies/{id}/render   authenticated  ?ticketId= fills {{agent.name}}, {{ticket.number}},
                                                     {{ticket.subject}}, {{customer.name}}, {{customer.number}}
                                                     and increments usageCount

Rules:
- Dashboard: AsNoTracking projections only, scoped by branch/department; counts myOpen,
  myPendingCustomer, myAtRisk, myResolvedToday, unassignedInScope, escalatedInScope,
  openTasks, overdueTasks, unreadNotifications.
- Tasks for someone else (create, or list by assignee id) need tickets.assign -> 403 FORBIDDEN.
  A task is visible/editable to its assignee, its creator or tickets.assign holders; otherwise
  404 TASK_NOT_FOUND.
- Reminders: recurring job every minute sends one task.reminder notification per task
  (remindAt <= now, not sent, not completed); changing remindAt re-arms it.
- Quick replies: personal (owner) or shared (no owner). Creating/editing/deleting shared
  replies needs quickreplies.manage -> 403 FORBIDDEN; someone else's personal reply -> 404
  QUICK_REPLY_NOT_FOUND. Shortcut stored upper-case without "/", returned as "/SHORTCUT".
- Messages localized (en/ar).
```

---

## Acceptance criteria

```
- [ ] Dashboard widgets use dedicated lightweight query projections (MyTickets, MyTasks, SlaAtRisk, RecentCustomers, UnreadMessages, PendingEscalations).
- [ ] Dashboard never loads full ticket aggregates.
- [ ] Tasks: create, update, complete, due date, reminder; optional ticket/customer link.
- [ ] Reminders fire via background job -> Notifications.
- [ ] Quick replies: personal vs shared (by department), categories, placeholders, localized variants.
- [ ] Mentions create in-app notifications and appear in the ticket timeline.
- [ ] Widget counters refresh in near real-time.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** Ticket Management (feature 02, `57e52f8`), CU-01, Phase 3 notifications.
- **Depends on code areas or other stories:** `TicketQueries.ProjectListAsync`, `NotificationSender`, `IAccessScopeProvider`, `AddRecurringRequest<T>` job runner, `AgentTask` / `QuickReply` entities (added in `57e52f8`).

## Extra notes (optional)

- As-built intake written after `customer-support-crm-api` commit `b81ba28` (only the `Features/Dashboard/AgentWorkspaceSlices.cs` part; the SLA part of that commit is feature 05).
- @mentions are implemented in ticket messages (feature 02), not in this slice.

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- SLA policies, assignment and escalation rules (feature 05).
