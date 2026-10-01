# Story intake

- Folder: `.squad/stories/frontend/task-past-date-warning/intake.md`

---

## Feature

- **Feature name (display):** Frontend (Agent Dashboard: tasks)
- **Feature slug (folder under `plans/`):** `frontend`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-26`
- **Work item type:** `Bug`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `frontend`, `bug`, `tasks`

---

## Title

```
Warn when a task due date or reminder is already in the past
```

---

## Description

```
QA round 2 (N6). A task saved with a due date one hour ago is accepted silently and is
"Overdue" at once. A reminder time in the past is also accepted; SendTaskRemindersHandler
(AgentWorkspaceSlices.cs ~272, RemindAt <= now) then sends it within a minute.
Past dates stay allowed (back-dating is legitimate); the dialog should just say so.
```

---

## Acceptance criteria

```
- [x] A past due date shows "This date is already past, so the task will be overdue right away." (warning colour).
- [x] A past reminder replaces the reminder hint with "...the reminder will be sent right away."
- [x] Future dates show the normal hints; en and ar keys exist.
- [x] npx ng build passes with 0 errors and 0 warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** none
- **Depends on code areas or other stories:** found in the round 2 manual QA pass, [../../../qa/2026-10-01-manual-qa-report-round2.md](../../../qa/2026-10-01-manual-qa-report-round2.md).

## Technical hints (optional)

- `features/dashboard/task-dialog.component.ts`; dates use `datetime-local` and `localInputToIso` (`dashboard.api.ts` 83).

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/ or e2e/.
- Replacing `datetime-local` with a Material picker (see plan 52 open question).
