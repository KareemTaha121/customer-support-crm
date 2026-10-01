# Story intake

- Folder: `.squad/stories/agent-dashboard/task-scope-and-reference-validation/intake.md`

---

## Feature

- **Feature name (display):** Agent Dashboard
- **Feature slug (folder under `plans/`):** `agent-dashboard`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-02`
- **Work item type:** `Bug`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `bug`, `tasks`

---

## Title

```
Tasks: enforce scope on ticket/customer filters and validate task references
```

---

## Description

```
ListTasksHandler (Features/Dashboard/AgentWorkspaceSlices.cs ~103-140): when ticketId or
customerId is given, the assignee filter is dropped and no branch/department scope is applied,
so GET /tasks?ticketId= / ?customerId= returns every assignee's tasks for any ticket/customer id.

Save task (create/update): an unknown or out-of-scope ticketId / customerId is not checked and
fails as a 500 at the foreign key.

Fix:
- With ticketId/customerId, return tasks only when the ticket/customer is in the caller's
  AccessScope; otherwise 404 (TICKET_NOT_FOUND / CUSTOMER_NOT_FOUND).
- Without tickets.assign, a caller still sees only their own tasks for that ticket/customer.
- Create/update validates ticketId/customerId exist and are in scope -> 404 with the same codes.
```

---

## Acceptance criteria

```
- [ ] GET /tasks?ticketId= for an out-of-scope or unknown ticket returns 404.
- [ ] Callers without tickets.assign never see other assignees' tasks.
- [ ] Creating/updating a task with an unknown ticket/customer id returns 404, not 500.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** none
- **Depends on code areas or other stories:** found while writing the as-built plans 20–36 (see `.squad/HANDOFF.md`, "Probable backend bugs").

## Technical hints (optional)

- Repo root: `customer-support-crm-api/` (branch `develop`). .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
