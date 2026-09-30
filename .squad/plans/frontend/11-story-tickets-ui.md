# Story 11 — Tickets UI (Story: FE-04)

## Prerequisites

- Story 08 completed: [08-story-core-platform-shell.md](08-story-core-platform-shell.md) (`ApiService`, `ApiError`, i18n, permissions, `StaffHubService`, shared states/forms).
- Story 09 completed: [09-story-authentication-staff-and-portal.md](09-story-authentication-staff-and-portal.md) (staff session, `AuthService.currentUser()`).
- Conventions and ownership: [00-overview.md](00-overview.md). This story edits **only** `src/app/features/tickets/**` and `public/i18n/tickets/{en,ar}.json`.
- Cross-feature contracts used **as-is** (owned by other stories, **do not edit**): `features/ai/ticket-ai-panel.component.ts` (Story 16) and `features/dashboard/quick-reply-picker.component.ts` (Story 13).

---

## Story Goal

1. Agents browse tickets at `/tickets` with filters (status, priority, category, assignee, created date range, search), server paging and sorting.
2. Agents create a ticket at `/tickets/new` (customer autocomplete, subject, description, category, priority, channel Agent/Phone, optional assignee, tags).
3. The ticket details page `/tickets/:id` shows the conversation (public replies vs **visually distinct** internal notes), a reply box with attachments, quick replies and AI suggestions, the attachments list (download/delete), a side panel (customer, SLA due dates and breach flags, assignee, category/priority edit, branch/department transfer) and a history tab.
4. Only the actions valid for the current status are offered (`TicketResponse.allowedStatuses` + escalate rules), each gated by its permission; domain errors are shown.
5. The page refreshes when the hub pushes `ticketUpdated` for the shown ticket; the list refreshes on any `ticketUpdated`.
6. Category admin at `/tickets/categories`: hierarchical list, create/edit (English + Arabic name, parent, default department, default priority, sort order, active).

Not in scope: mentions picker (`mentionedUserIds` is sent empty), satisfaction feedback (portal only), unit/e2e tests.

---

## Context — Read These Files First

1. `customer-support-crm-api/src/CustomerSupportCrm.Contracts/Tickets/TicketContracts.cs` — every request/response record: `CreateTicketRequest`, `UpdateTicketRequest`, `AssignTicketRequest`, `ChangeTicketStatusRequest`, `EscalateTicketRequest`, `TransferTicketRequest`, `AddTicketMessageRequest`, `TicketListItemResponse`, `TicketResponse` (note `allowedStatuses`), `TicketSlaResponse`, `TicketMessageResponse`, `TicketHistoryResponse`, `TicketCategoryRequest/Response`.
2. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Tickets/TicketQuerySlices.cs` ~lines 22–60 — `ListTicketsQuery` filters: `status` (comma list or `active`), `priority` (comma list), `assignee` (`me`/`unassigned`/id), `categoryId`, `createdFrom/To`, `sortBy` ∈ `createdAt|updatedAt|priority|dueAt|number`; ~lines 225–254 — `GET /tickets`, `/tickets/{id}`, `/{id}/messages`, `/{id}/history`.
3. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Tickets/TicketCommandSlices.cs` ~lines 38–49 (create validation: subject ≤ 300, description ≤ 20 000, channel only `Agent`/`Phone`, tags ≤ 20), ~lines 204–222 (`TicketAssignment`: self-assign needs `tickets.update`, others/unassign needs `tickets.assign`; `AGENT_NOT_ELIGIBLE`), ~lines 264–292 (status `"Reopen"`), ~lines 294–311 (escalate reason ≤ 500), ~lines 329–384 (routes + permissions: transfer = `tickets.assign`, escalate = `tickets.escalate`, delete = `tickets.delete`).
4. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Tickets/TicketMessageSlices.cs` ~lines 64–75 (message body ≤ 50 000, ≤ 10 attachments), ~lines 164–204 — `POST /tickets/{id}/messages`, `GET|POST /attachments`, `GET|DELETE /attachments/{attachmentId}`. Files are uploaded first, then referenced by `attachmentIds`.
5. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Tickets/TicketCategorySlices.cs` ~lines 19–106 — `GET /ticket-categories?includeInactive=`, `POST`, `PUT /{id}` (`tickets.categories_manage`); no delete (deactivate via `isActive`).
6. `customer-support-crm-api/src/CustomerSupportCrm.Domain/Tickets/Ticket.cs` ~lines 17–31 — error codes (`INVALID_STATUS_TRANSITION`, `TICKET_CLOSED`) and the `Transitions` table; ~lines 283–310 — reopen only from Resolved/Closed, escalation blocked when Resolved (and Closed via `EnsureNotClosed`).
7. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Tickets/Common/TicketQueries.cs` ~lines 14–21 (`TicketErrors`), ~lines 138–144 (`allowedStatuses` includes `"Reopen"`), line 244 (`DownloadPath` = `/api/v1/tickets/{id}/attachments/{attachmentId}`).
8. `customer-support-crm-api/src/CustomerSupportCrm.Domain/Tickets/TicketEnums.cs` and `TicketRecords.cs` ~lines 125–137 (`TicketHistoryActions`: `created`, `status`, `assignee`, `priority`, `category`, `department`, `escalated`, `sla_warning`, `sla_breached`, `feedback`).
9. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Users/Administration/UserAdministration.cs` ~lines 106–135, 168–171 — `GET /users/lookup?search=&permission=` → `UserLookupResponse(id, displayName, email)`.
10. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Customers/CustomerProfileSlices.cs` ~lines 209–215, 330–337 — `GET /customers?search=&pageSize=` → `CustomerListItemResponse` (`Contracts/Customers/CustomerContracts.cs` ~line 35).
11. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Branches/BranchEndpoints.cs` ~lines 140–146 — `GET /branches` (any staff) → `BranchResponse` with `departments` (transfer dialog, category default department).
12. `customer-support-crm-api/src/CustomerSupportCrm.Application/Abstractions/Notifications/IRealtimeNotifier.cs` — `RealtimeEvents.TicketUpdated = "ticketUpdated"` (payload not typed; match `ticketId` or `id`).
13. `customer-support-crm-web/src/app/core/http/api.service.ts`, `core/http/api-error.ts`, `shared/form-errors.ts`, `core/interceptors/error.interceptor.ts` (`describeError`), `core/realtime/staff-hub.service.ts`, `shared/confirm-dialog.component.ts`, `features/auth/profile.page.ts` (form error style).

---

## Frontend Tasks

No backend changes required.

### 1 — Models and API

Create file: `customer-support-crm-web/src/app/features/tickets/tickets.models.ts`

- Interfaces mirroring the contracts above, `TicketStatus`/`TicketPriority` string unions, `TICKET_STATUSES`, `TICKET_PRIORITIES`, `TicketListQuery`, `statusPill()`/`priorityPill()`/`slaPill()` (map to `crm-pill--*`), `buildCategoryTree()` (flat list → ordered list with `depth`) and `categoryLabel(category, language)` (uses `nameAr` in Arabic when present).

Create file: `customer-support-crm-web/src/app/features/tickets/tickets.api.ts`

```ts
@Injectable({ providedIn: 'root' })
export class TicketsApi {
  list(query: TicketListQuery): Observable<Paged<TicketListItem>>;      // GET /tickets
  get(id), messages(id), history(id), attachments(id);                   // GET /tickets/{id}[/...]
  create(body, silent), update(id, body, silent);                        // POST/PUT /tickets
  assign(id, agentId), transfer(id, body), changeStatus(id, status), escalate(id, reason), delete(id);
  addMessage(id, body, silent), uploadAttachment(id, file), downloadAttachment(att), deleteAttachment(id, attId);
  categories(includeInactive), saveCategory(id | null, body, silent);    // /ticket-categories
  lookupUsers(search), searchCustomers(search), branches();              // pickers
}
```

### 2 — Routes

File: `customer-support-crm-web/src/app/features/tickets/tickets.routes.ts` — replace the placeholder, keep `TICKETS_ROUTES`:

```ts
export const TICKETS_ROUTES: Routes = [{
  path: '', resolve: { i18n: translationResolver('tickets') }, children: [
    { path: '', canActivate: [requirePermission(Permissions.ticketsView)], loadComponent: ... TicketListPage },
    { path: 'new', canActivate: [requirePermission(Permissions.ticketsCreate)], loadComponent: ... TicketCreatePage },
    { path: 'categories', canActivate: [requirePermission(Permissions.ticketCategoriesManage)], loadComponent: ... TicketCategoriesPage },
    { path: ':id', canActivate: [requirePermission(Permissions.ticketsView)], loadComponent: ... TicketDetailsPage },
  ],
}];
```

### 3 — List page

Create file: `customer-support-crm-web/src/app/features/tickets/ticket-list.page.ts` (+ `.html`)

- Filter bar: search (debounced 300 ms), status multi-select (plus "Active"), priority multi-select, category select (tree), assignee (`me` / `unassigned` / agent picker), created from/to (native date inputs → ISO start/end of day). Filters are mirrored to query params so reload/back keeps them.
- `mat-table` columns: number, subject + customer, status pill, priority pill, category, assignee, SLA pill + resolution due, created. `mat-sort` on `number`, `priority`, `dueAt`, `createdAt`, `updatedAt`; `mat-paginator` (10/25/50/100).
- Header action "New ticket" behind `*appHasPermission="'tickets.create'"`. Row click → `/tickets/:id`.
- Realtime: `hub.on(RealtimeEvents.ticketUpdated)` debounced 1 s → reload current page.

### 4 — Shared pickers

Create file: `customer-support-crm-web/src/app/features/tickets/agent-picker.component.ts` — `mat-autocomplete` over `GET /users/lookup?search=&permission=tickets.update`; inputs `label`, `value`; output `picked` (`UserLookup | null`).

### 5 — Create page

Create file: `customer-support-crm-web/src/app/features/tickets/ticket-create.page.ts`

- Reactive form: `customerId` (customer autocomplete over `GET /customers?search=`, `?customerId=` query param preselects), `subject` (required, ≤ 300), `description` (≤ 20 000), `categoryId`, `priority` (default `Medium`; picking a category with `defaultPriority` applies it), `channel` (`Agent`/`Phone`), `tags` (chips, ≤ 20), `assignedAgentId` (only with `tickets.assign`).
- Submit with `{ silent: true }`; `applyServerErrors` + toast for unmatched; navigate to the new ticket.

### 6 — Details page

Create files: `ticket-details.page.ts` + `ticket-details.page.html` + `ticket-details.page.scss`, `ticket-conversation.component.ts`, `ticket-side-panel.component.ts`, `ticket-history.component.ts`, `ticket-dialogs.ts` (escalate reason, edit subject/description/tags, transfer), all under `customer-support-crm-web/src/app/features/tickets/`.

- Loads `ticket`, `messages`, `attachments` in parallel; `history` lazily when the tab opens. 404 → error state with a back link.
- Header: number, subject, status/priority pills, escalation level; actions menu built from `ticket.allowedStatuses`:
  - `Resolved` → "Resolve", `Closed` → "Close", `Reopen` → "Reopen", other statuses → "Move to …" (`tickets.update`).
  - "Escalate" when status is not `Resolved`/`Closed` (`tickets.escalate`) → reason dialog (required, ≤ 500).
  - "Edit" (subject/description/tags, `tickets.update`), "Transfer" (`tickets.assign`), "Delete" (`tickets.delete`, confirm, back to list).
- Conversation tab: messages oldest first; agent/customer/system authors styled differently; internal notes on a warning-tinted card with a lock icon and "Internal note" label. Reply box: public reply / internal note toggle, textarea, file picker (uploads immediately to `/attachments`, shows chips with remove = delete), `<app-quick-reply-picker (selected)>` inserts at the end, send (`tickets.update`). Closed tickets show a hint that replying keeps the ticket closed (server enforces rules).
- Attachments tab: list with size, uploader, date, public/internal pill, download (`ApiService.download(downloadUrl)` + `saveBlob`), delete (confirm, `tickets.update`).
- History tab: `ticket-history.component.ts` timeline with action labels (`tickets.history.actions.*`), old → new values (status/priority values translated).
- Side panel: customer card (link to `/customers/:id`), SLA block (policy, first response due/responded/breached, resolution due/resolved/breached, state pill), assignee (agent picker when `tickets.assign`, "Assign to me" with `tickets.update`, "Unassign" with `tickets.assign`), category + priority selects (PUT with current subject/description/tags, `tickets.update`), branch/department, channel, tags, created/updated.
- `<app-ticket-ai-panel [ticketId]="ticket.id" (replySuggested)="conversation.insert($event)" (categorySuggested)="applySuggestion($event)">` below the side panel.
- Realtime: `hub.on<unknown>(RealtimeEvents.ticketUpdated)` filtered to this ticket id → reload ticket + messages + attachments (history if loaded).

### 7 — Category admin

Create files: `customer-support-crm-web/src/app/features/tickets/ticket-categories.page.ts`, `ticket-category-dialog.component.ts`

- Table (tree order, indented by depth): name, Arabic name, parent, default priority, sort order, active pill; "Show inactive" toggle (`includeInactive=true`); create/edit dialog (name required ≤ 150, nameAr ≤ 150, parent select excluding itself and its descendants, default department from `GET /branches`, default priority, sort order, active). Server errors via `applyServerErrors`.

### 8 — Translations

Create files: `customer-support-crm-web/public/i18n/tickets/en.json`, `ar.json` — identical key sets: `list`, `filters`, `columns`, `fields`, `status.*`, `priority.*`, `channel.*`, `sla.*`, `authorType.*`, `actions.*`, `details.*`, `reply.*`, `attachments.*`, `history.actions.*`, `dialogs.*`, `create.*`, `categories.*`, `messages.*`.

---

## Edge Cases & Failure Modes

- Invalid transition raced by another agent (`INVALID_STATUS_TRANSITION`, `TICKET_CLOSED`) → the global snackbar shows the server message; the details page reloads the ticket so `allowedStatuses` is fresh (`ticket-details.page.ts` `runAction`).
- Assigning someone else without `tickets.assign` → picker hidden; the server returns `ASSIGN_FORBIDDEN` if forced. `AGENT_NOT_ELIGIBLE` → snackbar, assignee unchanged (`ticket-side-panel.component.ts`).
- Ticket deleted/out of scope while open → `TICKET_NOT_FOUND` (404) → error state (`ticket-details.page.ts` `load`).
- Upload too large / blocked type (413/400) → snackbar; the pending chip is not added (`ticket-conversation.component.ts` `onFiles`). More than 10 pending files → the extra ones are ignored with a toast.
- Uploaded but unsent files stay on the ticket as internal attachments → removing the chip deletes them via `DELETE /attachments/{id}`.
- `ticketUpdated` payload shape is untyped → compare `payload.ticketId ?? payload.id` (string) with the current id; the list refresh is debounced so bursts reload once.
- Category with `nameAr` null in Arabic → falls back to `name` (`categoryLabel`).
- Category edit: a parent cannot be itself or a descendant (filtered in the dialog) to avoid cycles.
- Date range filter: `createdTo` uses end of day in local time converted to ISO so the whole day is included.
- Empty lists (no messages / attachments / history) → `<app-empty-state>`.

## Test Plan

Out of scope for this story (build-level verification only, per [00-overview.md](00-overview.md)). Manual smoke:

1. List: filter by status + priority + assignee "me", sort by priority, change page size; reload keeps filters.
2. Create a ticket for a searched customer with category → lands on details.
3. Details: send a public reply with an attachment, send an internal note (distinct styling), download and delete an attachment.
4. Resolve → Close → Reopen; Escalate with reason; history shows each step.
5. Assign to me, reassign via picker, change category/priority from the side panel.
6. Categories: create a child category with an Arabic name; switch to Arabic and verify labels and RTL layout.

## Verification Steps

1. **Frontend builds:** `cd customer-support-crm-web && npx ng build` — no errors or warnings in `src/app/features/tickets/**`.
2. **Regression:** `/tickets/categories` sidenav link opens the category admin page; other features untouched.

## Done Criteria

- [ ] Every endpoint in `Application/Features/Tickets` has a UI entry point (list, get, messages, history, create, update, assign, transfer, status/reopen, escalate, delete, message, attachments list/upload/download/delete, categories list/create/update).
- [ ] Only actions valid for the current status are offered; domain errors are shown.
- [ ] Internal notes are visually distinct from public replies.
- [ ] en/ar translations with identical keys; RTL-safe styles (logical properties).
- [ ] Realtime `ticketUpdated` refreshes the details page and the list.
- [ ] AI panel and quick reply picker are embedded through their contracts.
