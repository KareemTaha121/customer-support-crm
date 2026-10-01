# Story 27 — Agent workspace: dashboard, tasks & reminders, quick replies (Story: AD-01)

> As-built plan: written after implementation in `customer-support-crm-api` commit `b81ba28` (only `Features/Dashboard/AgentWorkspaceSlices.cs`, `Contracts/Dashboard/DashboardContracts.cs` and the reminder job line; the SLA part of that commit is feature 05). The `AgentTask` / `QuickReply` entities, their EF configuration and DbSets were added earlier in `57e52f8` (feature 02). Paths and line numbers refer to `b81ba28` unless a line says otherwise.

## Prerequisites

- Ticket Management (feature 02, `57e52f8`): `Ticket`, `TicketStatus`, SLA fields on the ticket, `TicketQueries.ProjectListAsync` (`Features/Tickets/Common/TicketQueries.cs` line 55) and `TicketLink` (line 242), ticket-message @mentions.
- Customer Management: [../customer-management/23-story-customer-profiles-and-contacts.md](../customer-management/23-story-customer-profiles-and-contacts.md) — `db.Customers`.
- Phase 3 (`65c74a3`): `NotificationSender` (`Features/Notifications/NotificationSlices.cs` lines 24–38), `IAccessScopeProvider`, the recurring-request job runner (`AddRecurringRequest<T>`), the `/hubs/staff` `notificationCreated` push.
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

1. `GET /dashboard/agent` returns, in one call, the agent's counters and short lists: my tickets by due date, SLA at risk, pending escalations, recent customers, my open tasks.
2. Agents keep personal (or, with `tickets.assign`, delegated) tasks linked to a ticket or customer, with due date and a reminder that becomes an in-app notification.
3. Agents use quick replies — personal or shared — and render them with ticket/customer/agent placeholders filled.

**Deviations from the intake (the code is authoritative):**

| Feature spec | As built |
|---|---|
| `Features/AgentWorkspace/{Dashboard,Tasks,QuickReplies,Collaboration}/` | One file `Features/Dashboard/AgentWorkspaceSlices.cs`; entities in `Domain/Tickets/AgentWorkspace.cs` |
| Separate widget queries (`MyTickets`, `SlaAtRisk`, …) | One `GetAgentDashboardQuery` returning all widgets (lines 25–74); each widget is its own `AsNoTracking` projection |
| `UnreadMessages` widget | `Counts.UnreadNotifications` — unread in-app notifications, not unread ticket/chat messages |
| `Reminder` entity | `RemindAt` / `ReminderSentAt` columns on `AgentTask` |
| Quick replies shared **by department** | Shared = global (`OwnerId == null`), visible to every staff user |
| Quick reply categories | `CategoryId` is a free `Guid?`; no category entity, no validation |
| Localized variants | One `Language` (`en`/`ar`) per reply; no grouping of variants |
| `TicketWatcher`, watch/unwatch, handover | **Not built** |
| `Mention` entity, mention slice | Mentions are a `MentionedUserIds` `uuid[]` column on ticket messages (feature 02, `57e52f8`); `TicketEventHandlers.cs` lines 163–166 send `ticket.mentioned` notifications |
| Widget counters refresh in near real time | Dashboard is pull-only. Only notifications are pushed (`notificationCreated` on `/hubs/staff`); the client must re-fetch |
| Permissions for tasks / quick replies | No endpoint permission; any authenticated staff user. Rules inside the handlers use `tickets.assign` and `quickreplies.manage` |

**Not in scope:** SLA engine, assignment and escalation rules (feature 05). **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Domain/Tickets/AgentWorkspace.cs` (`57e52f8`) — `AgentTask` lines 7–85 (`TitleMaxLength = 300`, `INVALID_TASK`, `Create`, `Update` re-arms the reminder when `RemindAt` changes (71–75), `Reassign`, `Complete` (idempotent), `Reopen`, `MarkReminderSent`); `QuickReply` lines 88–155 (`TitleMaxLength = 150`, `ShortcutMaxLength = 30`, `BodyMaxLength = 10_000`, `INVALID_QUICK_REPLY`, `Create(ownerId)`, `Update` stores the shortcut upper-case without `/`, `RecordUse`).
2. `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/SlaConfiguration.cs` (`57e52f8`) — `agent_tasks` lines 87–103 (indexes `(AssigneeId, CompletedAt, DueAt)` and `(RemindAt, ReminderSentAt)`; assignee FK cascade, ticket/customer FK set null), `quick_replies` lines 105–119 (index `(OwnerId, Shortcut)`, owner FK cascade).
3. `src/CustomerSupportCrm.Application/Abstractions/Persistence/IApplicationDbContext.Automation.cs` — lines 17–19 (`AgentTasks`, `QuickReplies`).
4. `src/CustomerSupportCrm.Domain/Roles/Permissions.cs` — line 9 `tickets.view`, 12 `tickets.assign`, 33 `quickreplies.manage` (manager defaults).
5. `src/CustomerSupportCrm.Domain/Notifications/Notification.cs` — line 69 `ticket.mentioned`, 73 `task.reminder`.

---

## Backend Tasks

### 1 — Contracts

Create file: `src/CustomerSupportCrm.Contracts/Dashboard/DashboardContracts.cs` — `AgentDashboardCounts` (5–14: `MyOpen`, `MyPendingCustomer`, `MyAtRisk`, `MyResolvedToday`, `UnassignedInScope`, `EscalatedInScope`, `OpenTasks`, `OverdueTasks`, `UnreadNotifications`), `RecentCustomerResponse` (16), `TaskResponse` (18–31), `AgentDashboardResponse` (33–39, ticket lists reuse `TicketListItemResponse`), `TaskRequest` (42), `QuickReplyRequest` (45), `QuickReplyResponse` (47–56, `Shared`, `CanEdit`, `UsageCount`), `RenderedQuickReplyResponse` (58).

### 2 — Dashboard

`GetAgentDashboardHandler` (lines 28–74), deps `IApplicationDbContext, IAccessScopeProvider, ICurrentUser, TimeProvider`:

- Base queries (40–44): `scoped` = tickets `AsNoTracking().WhereInScope(scope)`; `active` = not `Resolved`/`Closed`; `mine` = assigned to me; `atRisk` = breached or warned on first response or resolution; `tasks` = my open tasks.
- Nine `CountAsync` calls (46–55); "resolved today" uses the UTC day start (37).
- Lists (57–70), `ListSize = 10`: my tickets by `ResolutionDueAt` (nulls last), SLA at risk, escalations by `EscalatedAt` desc — all through `TicketQueries.ProjectListAsync`; recent customers (top 8 by my tickets' last update, joined to `Customers`); my tasks by `DueAt`.

### 3 — Tasks

- `TaskQueries` (78–97): `TASK_NOT_FOUND`; `Project` adds assignee name, ticket number, customer name by sub-select.
- `ListTasksQuery` + handler (101–144): `assignee=<guid>` needs `tickets.assign` (403 `FORBIDDEN`); with no assignee but a `ticketId`/`customerId`, tasks of every assignee for that ticket/customer are returned; `status` `completed`/`all`/default open; open first, by due date, max 200. No validator.
- `SaveTaskCommand` + validator (146–155: title ≤ 300, notes ≤ 4000) + handler (157–199): assigning to someone else needs `tickets.assign`; `LoadOwnAsync` (187–198) allows assignee, creator (`CreatedBy`) or `tickets.assign`, else 404.
- `ChangeTaskCommand` + handler (202–224): `complete`, `reopen`, anything else deletes (hard delete).
- `SendTaskRemindersCommand` + handler (227–247): up to 500 due reminders per run; `NotificationSender.Notify(assignee, task.reminder, "Reminder: …", ticket link or "/tasks", {taskId, title})`; `MarkReminderSent`. Registered in `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` line 86: `services.AddRecurringRequest<SendTaskRemindersCommand>(TimeSpan.FromMinutes(1));`.

### 4 — Quick replies

- `ListQuickRepliesQuery` + handler (251–277): shared + mine; optional `language`; search on upper-cased title, shortcut prefix (leading `/` stripped) or body; ordered by `UsageCount` desc then title; max 100.
- `QuickReplyMapping` (279–303): `QUICK_REPLY_NOT_FOUND`; `ToResponse` returns `/SHORTCUT`; `LoadEditableAsync` — another user's personal reply → 404, shared without `quickreplies.manage` → 403 `FORBIDDEN`.
- `SaveQuickReplyCommand` + validator (305–316: title ≤ 150, shortcut ≤ 30 matching `^/?[A-Za-z0-9_-]+$`, body ≤ 10 000, language `en|ar`) + handler (318–346): creating a shared reply needs `quickreplies.manage`.
- `DeleteQuickReplyCommand` (348–357): hard delete.
- `RenderQuickReplyQuery` + handler (363–395): `{{agent.name}}` always; with `ticketId` (only if the ticket is in the caller's scope) `{{ticket.number}}`, `{{ticket.subject}}`, `{{customer.name}}`, `{{customer.number}}`; increments `UsageCount` and saves.

### 5 — Endpoints

`AgentWorkspaceEndpoints` (399–465), tag `Agent workspace`:

| Route | Permission | Name |
|---|---|---|
| `GET /dashboard/agent` | `tickets.view` | `GetAgentDashboard` |
| `GET /tasks`, `POST /tasks` (201, `Location: /api/v1/tasks`) | authenticated | `ListTasks`, `CreateTask` |
| `PUT /tasks/{id}`, `DELETE /tasks/{id}` | authenticated | `UpdateTask`, `DeleteTask` |
| `POST /tasks/{id}/complete`, `POST /tasks/{id}/reopen` | authenticated | `CompleteTask`, `ReopenTask` |
| `GET /quick-replies`, `POST /quick-replies` (201, `Location: /api/v1/quick-replies`) | authenticated | `ListQuickReplies`, `CreateQuickReply` |
| `PUT /quick-replies/{id}`, `DELETE /quick-replies/{id}` | authenticated | `UpdateQuickReply`, `DeleteQuickReply` |
| `POST /quick-replies/{id}/render?ticketId=` | authenticated | `RenderQuickReply` |

### 6 — Persistence, DI, localization

- No new tables in `b81ba28` (entities from `57e52f8`); the migration is `20260930104602_AddSupportOperations.cs` (`678ea67`), tables `quick_replies` and `agent_tasks`.
- Slices and endpoints are found by assembly scanning; only the reminder job line is new DI.
- `Messages.resx` (at `0936711`): `TASK_NOT_FOUND` (186), `QUICK_REPLY_NOT_FOUND` (189). `Messages.ar.resx`: `TASK_NOT_FOUND` (234), `INVALID_TASK` (237), `QUICK_REPLY_NOT_FOUND` (240), `INVALID_QUICK_REPLY` (243).
- `docs/endpoints.md` lines 111–123 list the routes (with the notification endpoints from Phase 3).

---

## Edge Cases & Failure Modes

- **Other agent's task** — 404 `TASK_NOT_FOUND` (not 403) unless creator or `tickets.assign`.
- **Task linked to an unknown ticket/customer id** — not validated; the FK fails on save and returns a generic 500 (`GlobalExceptionHandler` maps only unique violations to 409).
- **Task list by ticket/customer** — not scope-checked and not limited to the caller: any staff user can list all tasks linked to any ticket or customer id.
- **Reminder in the past** — sent on the next run (≤ 1 minute). Completed tasks never remind; changing `remindAt` re-arms; reopening does not reset `ReminderSentAt`, so an already-sent reminder is not sent again.
- **Shared reply edit without permission** — 403 `FORBIDDEN`; another user's personal reply — 404.
- **Render with an out-of-scope or unknown ticket** — ticket placeholders are left as-is, no error.
- **Shortcut** — `/greet` and `greet` are stored as `GREET`; no uniqueness constraint.
- **Dashboard for a user without scopes** — ticket counters are 0; "my" counters still filter by scope first.

---

## Test Plan

Test projects are **out of scope** (`tests/` must not be modified). No tests are added, changed or removed by this story, and no test in the repo (at `2956767`) covers the dashboard, tasks, reminders or quick replies.

---

## Verification Steps

1. **Backend builds:** `dotnet build` — 0 warnings, 0 errors.
2. **Dashboard:** `curl https://localhost:<port>/api/v1/dashboard/agent -H "Authorization: Bearer $TOKEN"` → `counts` plus five lists (≤ 10 items, recent customers ≤ 8).
3. **Task + reminder:** `POST …/tasks` with `{"title":"Call back","remindAt":"<now + 1 min>"}` → 201; within ~2 minutes `GET …/notifications` shows a `task.reminder` notification and `/hubs/staff` emits `notificationCreated`. `POST …/tasks/{id}/complete` → 200; `GET …/tasks?status=completed` lists it.
4. **Delegation:** an agent without `tickets.assign` posts `"assigneeId":"<other user>"` → 403 `FORBIDDEN`.
5. **Quick reply:** `POST …/quick-replies` `{"title":"Greeting","shortcut":"/greet","body":"Hello {{customer.name}}, ticket {{ticket.number}}. — {{agent.name}}","language":"en","shared":false}` → 201 with `shortcut: "/GREET"`; `POST …/quick-replies/{id}/render?ticketId=<id>` → placeholders filled, `usageCount` +1. `"shared":true` as an agent → 403.
6. **Localization:** `PUT …/tasks/<unknown guid>` with `Accept-Language: ar` → Arabic `TASK_NOT_FOUND`.

---

## Done Criteria

- [x] Dashboard widgets are dedicated `AsNoTracking` projections (my tickets, SLA at risk, pending escalations, recent customers, my tasks, counters).
- [x] Dashboard never loads full ticket aggregates.
- [x] Tasks: create, update, complete, reopen, delete, due date, reminder, optional ticket/customer link.
- [x] Reminders fire via a recurring background job → in-app notifications.
- [x] Quick replies: personal vs shared, placeholders, `en`/`ar` language, usage count.
- [ ] Shared by department, category entity, localized variants — not built.
- [ ] Unread messages widget — counts unread notifications instead.
- [x] Mentions create in-app notifications (built in feature 02 ticket messages, not in this slice).
- [ ] Ticket watchers/followers and handover — not built.
- [ ] Widget counters refresh in near real time — pull only (notifications are pushed).
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/` by this story.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding.**
