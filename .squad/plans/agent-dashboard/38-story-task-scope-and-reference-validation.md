# Story 38 — Tasks: enforce scope on ticket/customer filters and validate task references (Bug: BUG-02)

> Fix plan, implemented in `customer-support-crm-api` commit `7058b62` (`develop`). Paths and line numbers refer to that commit.
> Intake: [../../stories/agent-dashboard/task-scope-and-reference-validation/intake.md](../../stories/agent-dashboard/task-scope-and-reference-validation/intake.md)

## Prerequisites

- Story 27 — [27-story-agent-workspace.md](27-story-agent-workspace.md): `AgentTask`, `ListTasksHandler`, `SaveTaskHandler`, `TaskQueries`.
- Story 21 — [../platform/21-story-organization-context-and-notifications.md](../platform/21-story-organization-context-and-notifications.md): `IAccessScopeProvider`, `WhereInScope`.
- `TicketErrors.TicketNotFound` (`Features/Tickets/Common/TicketQueries.cs` line 16) and `CustomerErrors.CustomerNotFound` (`Features/Customers/Common/CustomerQueries.cs` line 15). Both codes already have en/ar resx entries.
- All paths are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

Before the fix:

1. **The list leaked across scope.** `GET /tasks?ticketId=` or `?customerId=` dropped the assignee filter and applied no branch/department scope. Any staff user could list every assignee's tasks for any ticket or customer id. An explicit `assignee=me` was also ignored when those filters were present.
2. **A bad link caused a 500.** `POST /tasks` and `PUT /tasks/{id}` with an unknown `ticketId` or `customerId` failed at the foreign key.

After the fix:

- A `ticketId` or `customerId` filter requires the record to exist **and** be in the caller's scope. Otherwise the response is 404 `TICKET_NOT_FOUND` / `CUSTOMER_NOT_FOUND`, so out-of-scope records are not disclosed.
- Without `tickets.assign`, the list always contains only the caller's own tasks. With `tickets.assign` and a record filter, it shows every assignee's tasks on that record (the "team view" of a ticket). `assignee=me` is always honoured.
- Create checks both links. Update checks only links that **change**, so a task stays editable after its ticket or customer moves out of the caller's scope.

**Deviation from the intake:** none. The intake said "otherwise 404" for unknown and out-of-scope references alike, and that is what was built.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Application/Features/Dashboard/AgentWorkspaceSlices.cs`:
   - `TaskQueries` (79–115): `Project`, and the new `EnsureReferencesInScopeAsync` (103–114).
   - `ListTasksQuery` (122) and `ListTasksHandler` (124–172).
   - `SaveTaskHandler` (185–236).
2. `src/CustomerSupportCrm.Application/Common/Authorization/AccessScope.cs`: `WhereInScope` and `EnsureAccess`; the 404-not-403 convention.
3. `docs/endpoints.md` line 116: the tasks row.

---

## Backend Tasks

### 1 — Shared reference check

Add to `TaskQueries`:

```csharp
public static async Task EnsureReferencesInScopeAsync(IApplicationDbContext db, AccessScope scope, Guid? ticketId, Guid? customerId, CancellationToken ct);
```

- `ticketId` set → `db.Tickets.AsNoTracking().Where(id).WhereInScope(scope).AnyAsync()`; when false, throw `NotFoundException(TicketErrors.TicketNotFound, …)`.
- `customerId` set → the same on `db.Customers`. The soft-delete query filter also excludes deleted customers. When false, throw `NotFoundException(CustomerErrors.CustomerNotFound, …)`.
- Add `using CustomerSupportCrm.Application.Features.Customers.Common;`.

### 2 — List

Inject `IAccessScopeProvider scopes` into `ListTasksHandler`.

- `byRecord = ticketId or customerId is set`. When true, call `EnsureReferencesInScopeAsync` first.
- `assignee=<guid>` keeps requiring `tickets.assign`. Without it the response is still 403 `FORBIDDEN`.
- Otherwise the filter is `AssigneeId == me` when `assignee == "me"`, when there is no record filter, or when the caller lacks `tickets.assign`. That leaves exactly one case with no assignee filter: `tickets.assign` plus an in-scope record filter.
- The status filter, ordering and `Take(200)` are unchanged.

### 3 — Save

Inject `IAccessScopeProvider scopes` into `SaveTaskHandler`.

- **Create:** call `EnsureReferencesInScopeAsync(scope, input.TicketId, input.CustomerId)` before `AgentTask.Create`.
- **Update:** after `LoadOwnAsync`, pass each link only when it differs from the stored value (`input.TicketId != task.TicketId ? input.TicketId : null`, and the same for the customer). Clearing a link (`null`) needs no check.

### 4 — Docs

In `docs/endpoints.md`, the `GET, POST /tasks` row now says: own tasks; `ticketId`/`customerId` must be in scope, else 404; with `tickets.assign` they list every assignee.

**No changes to:** contracts, domain, migrations, resx (existing codes), or the frontend. `dashboard.api.ts` calls `GET /tasks` with `status` only, so its behaviour is unchanged.

---

## Test Plan

Test projects are **out of scope**: `tests/` must not be modified. No existing test covers tasks.

---

## Verification Steps

1. **Build:** in `customer-support-crm-api/`, run `dotnet build`. At `7058b62`: 0 warnings, 0 errors.
2. **Own list unchanged:** an agent calls `GET /api/v1/tasks` and sees only their own open tasks.
3. **Out-of-scope ticket:** an agent scoped to branch A calls `GET /tasks?ticketId=<ticket in branch B>` and gets 404 `TICKET_NOT_FOUND`. A random Guid gets the same response.
4. **No tickets.assign:** an agent calls `GET /tasks?ticketId=<in-scope ticket that has tasks for other agents>` and sees only their own tasks.
5. **Supervisor:** a user with `tickets.assign` makes the same call as step 4 and sees every assignee's tasks. Adding `&assignee=me` returns only their own.
6. **Create with a bad link:** `POST /tasks` with `{"title":"x","ticketId":"<random guid>"}` returns 404 `TICKET_NOT_FOUND`, not 500. With `customerId` it returns 404 `CUSTOMER_NOT_FOUND`.
7. **Update keeps the old link:** move a task's ticket out of the agent's scope, then `PUT /tasks/{id}` with the same `ticketId` and a new title. The response is 200.
8. **Localization:** repeat step 6 with `Accept-Language: ar`; the message is in Arabic.

---

## Done Criteria

- [x] `GET /tasks?ticketId=` for an out-of-scope or unknown ticket returns 404, and the same holds for `customerId`.
- [x] Callers without `tickets.assign` never see other assignees' tasks.
- [x] Creating or updating a task with an unknown ticket or customer id returns 404, not 500.
- [x] Updating a task without changing its links works after the record leaves the caller's scope.
- [x] `docs/endpoints.md` is updated.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to the next bug story.**
