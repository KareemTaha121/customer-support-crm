# Story 13 — Agent dashboard UI (Story: FE-06)

## Prerequisites

- Story 08 completed: [08-story-core-platform-shell.md](08-story-core-platform-shell.md) (`ApiService`, i18n, `StaffHubService`, permissions, shared states/header/confirm).
- Story 09 completed: [09-story-authentication-staff-and-portal.md](09-story-authentication-staff-and-portal.md) (`AuthService.currentUser`, style precedent `features/auth/profile.page.ts`).
- Contract with Stories 11 and 12: `features/dashboard/quick-reply-picker.component.ts` keeps selector `app-quick-reply-picker`, class `QuickReplyPickerComponent` and output `selected` (reply body string). Conventions in [00-overview.md](00-overview.md).

---

## Story Goal

1. `/dashboard` (default landing page, no permission guard) greets the signed-in user and shows live counts and short lists from `GET /dashboard/agent`: my tickets, SLA at risk, pending escalations, recent customers, my open tasks — ticket rows link to `/tickets/:id`, customers to `/customers/:id`.
2. The dashboard refreshes (debounced) when the staff hub pushes `ticketUpdated` or `notificationCreated`.
3. `/dashboard/tasks`: my tasks with an open / completed / all filter, create/edit dialog (title, notes, due date, reminder, related ticket), complete/reopen, delete with confirmation.
4. `/dashboard/quick-replies`: search, create/edit/delete personal replies; shared replies can be created only with `quickreplies.manage` and edited only when the server says `canEdit`.
5. The picker (used in ticket/chat reply boxes) searches quick replies, renders the chosen one through `POST /quick-replies/{id}/render` (fills placeholders, counts the use) and emits the body.
6. en/ar translations with identical keys; RTL-safe CSS.

Not in scope: assigning tasks to other agents (`assigneeId`), customer-linked tasks, quick-reply categories (`categoryId` is sent as `null`, kept when editing).

---

## Context — Read These Files First

1. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Dashboard/AgentWorkspaceSlices.cs`
   - ~lines 28–78 `GetAgentDashboardHandler`: lists capped at 10, recent customers at 8; `mine` = active tickets assigned to me.
   - ~line 101 `ListTasksQuery(status = open|completed|all, assignee, ticketId, customerId)`; ~lines 146–200 `SaveTaskCommand` (title required, max length; notes ≤ 4000).
   - ~lines 204–226 `ChangeTaskHandler` (complete / reopen / delete).
   - ~lines 251–277 `ListQuickRepliesQuery(search, language)` — search matches title, body, shortcut prefix (leading `/` stripped).
   - ~lines 279–310 `QuickReplyMapping` — shortcut returned as `/UPPER`, `canEdit`, `shared`; errors `QUICK_REPLY_NOT_FOUND`, `FORBIDDEN`.
   - ~line 363 `RenderQuickReplyQuery(id, ticketId?)`; ~lines 399–464 endpoints: `GET /dashboard/agent` (requires `tickets.view`), `/tasks` GET/POST/PUT/DELETE + `POST /tasks/{id}/complete|reopen`, `/quick-replies` GET/POST/PUT/DELETE + `POST /quick-replies/{id}/render?ticketId=`.
2. `customer-support-crm-api/src/CustomerSupportCrm.Contracts/Dashboard/DashboardContracts.cs` — lines 5–58: `AgentDashboardCounts`, `RecentCustomerResponse`, `TaskResponse`, `AgentDashboardResponse`, `TaskRequest`, `QuickReplyRequest` (language `en|ar`, `shared`), `QuickReplyResponse`, `RenderedQuickReplyResponse`.
3. `customer-support-crm-api/src/CustomerSupportCrm.Contracts/Tickets/TicketContracts.cs` — lines 35–57 `TicketListItemResponse` (list rows; status/priority are enum strings from `Domain/Tickets/TicketEnums.cs`).
4. `customer-support-crm-web/src/app/core/realtime/staff-hub.service.ts` — `RealtimeEvents`, `hub.on<T>()`.
5. `customer-support-crm-web/src/app/core/http/api.service.ts` — `get`, `getPaged`, `post`, `put`, `delete`, `{ silent: true }`.
6. `customer-support-crm-web/src/app/shared/form-errors.ts` — `applyServerErrors`; `core/interceptors/error.interceptor.ts` ~line 33 `describeError`.
7. `customer-support-crm-web/src/app/features/auth/profile.page.ts` — component style precedent.

---

## Frontend Tasks

### 1 — Models and API

Create file: `customer-support-crm-web/src/app/features/dashboard/dashboard.models.ts` — interfaces mirroring the contracts above (`AgentDashboard`, `AgentDashboardCounts`, `DashboardTicket`, `RecentCustomer`, `AgentTask`, `TaskRequest`, `QuickReply`, `QuickReplyRequest`, `TaskStatusFilter = 'open' | 'completed' | 'all'`).

Create file: `customer-support-crm-web/src/app/features/dashboard/dashboard.api.ts` — `DashboardApi` (`providedIn: 'root'`): `agentDashboard()`, `tasks(status)`, `createTask`, `updateTask`, `completeTask`, `reopenTask`, `deleteTask`, `quickReplies(search)`, `createQuickReply`, `updateQuickReply`, `deleteQuickReply`, `renderQuickReply(id, ticketId?)`, `searchTickets(search)` (`GET /tickets?search=&pageSize=10` for the task dialog).

### 2 — Pages and dialogs

- `dashboard.page.ts` — greeting (`AuthService.currentUser().displayName`), KPI cards (my open, pending customer, at risk, resolved today, unassigned, escalated, open tasks, overdue tasks, unread notifications), ticket lists, recent customers, tasks widget (complete inline, link to tasks page). Hub events → `debounceTime(1500)` → silent reload.
- `tasks.page.ts` — filter chips, list, `TaskDialogComponent` (`task-dialog.component.ts`) using `datetime-local` inputs converted to ISO, ticket autocomplete.
- `quick-replies.page.ts` — search box (debounced), cards/table, `QuickReplyDialogComponent` (`quick-reply-dialog.component.ts`): title, shortcut (`^/?[A-Za-z0-9_-]+$`), body, language, shared toggle wrapped in `*appHasPermission="'quickreplies.manage'"` and disabled when editing.
- `quick-reply-picker.component.ts` — replace body: icon button + `mat-menu` with search input and list; optional `ticketId` input passed to render; `inject(TranslationService).load('dashboard')` in the constructor.

### 3 — Routes and translations

File: `dashboard.routes.ts` — `DASHBOARD_ROUTES = [{ path: '', resolve: { i18n: translationResolver('dashboard') }, children: [ '' → DashboardPage, 'tasks' → TasksPage, 'quick-replies' → QuickRepliesPage ] }]` (lazy `loadComponent`).

Create `public/i18n/dashboard/en.json` and `ar.json`.

---

## Edge Cases & Failure Modes

- User lacks `tickets.view` → `GET /dashboard/agent` returns 403: show greeting plus an info message and still offer tasks/quick replies (no global error: call with `silent`, render error state for other failures).
- Hub bursts (bulk updates) → debounced; refresh is silent (keeps content while loading).
- `datetime-local` has no timezone → converted via `new Date(value).toISOString()`; displayed back in local time.
- Shared reply with `canEdit=false` → edit/delete hidden. Server `FORBIDDEN` still surfaces via toast.
- Render call fails in the picker → emit the raw body so the agent is not blocked.

## Test Plan

Out of scope (build-level verification only).

## Verification Steps

1. `cd customer-support-crm-web && npx ng build` — no errors/warnings in `features/dashboard/**`.

## Done Criteria

- [ ] Dashboard shows live numbers and lists with links to tickets and refreshes on realtime events.
- [ ] Tasks and quick replies can be managed; shared replies gated by `quickreplies.manage`.
- [ ] Picker emits rendered body; en/ar + RTL.
