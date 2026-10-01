# Story 62 — Warn when a task due date or reminder is already past (Bug: BUG-26)

> Fix plan, implemented in `customer-support-crm-web` commit `b91dd22` (`main`). Paths and line numbers refer to that commit.
> Intake: [../../stories/frontend/task-past-date-warning/intake.md](../../stories/frontend/task-past-date-warning/intake.md)

## Prerequisites

- Frontend story 13 ([13-story-agent-dashboard-ui.md](13-story-agent-dashboard-ui.md)): `TaskDialogComponent`.
- Backend story 27 ([../agent-dashboard/27-story-agent-workspace.md](../agent-dashboard/27-story-agent-workspace.md)): `SendTaskRemindersHandler` runs every minute on `RemindAt <= now && ReminderSentAt == null` (`AgentWorkspaceSlices.cs` ~272).

---

## Story Goal

Past dates stay **allowed**: back-dating a follow-up is legitimate, and the API accepts it. Before the fix the dialog gave no hint that the task would be overdue at once, or that a past reminder would fire within a minute.

| Field | Past value | Future or empty value |
|---|---|---|
| Due | New hint (warning colour): "This date is already past, so the task will be overdue right away." | No hint (unchanged) |
| Reminder | Its hint becomes (warning colour): "This time is already past, so the reminder will be sent right away." | "You will get a notification at this time." (unchanged) |

**Deviation from the intake:** none. The first draft of the reminder text said "no reminder will be sent". That was corrected after reading the job, which sends past reminders straight away.

---

## Context — Read These Files First

1. `src/app/features/dashboard/task-dialog.component.ts`: template 66–77, `dueInPast` / `remindInPast` 141–143, `isPast` 205–208, style `.past-hint` 115.
2. `src/app/features/dashboard/dashboard.api.ts:83`: `localInputToIso`.
3. `public/i18n/dashboard/{en,ar}.json`: `dashboard.tasks.fields.dueInPastHint` and `remindInPastHint`.

---

## Frontend Tasks

1. Two signals built from the form controls:
   ```ts
   readonly dueInPast = toSignal(this.form.controls.dueAt.valueChanges.pipe(startWith(this.form.controls.dueAt.value), map(isPast)), { initialValue: false });
   ```
   `isPast(value)` converts the `datetime-local` value with `localInputToIso` and compares it to `Date.now()`.
2. Template: a `<mat-hint class="past-hint">` under Due when `dueInPast()`. The reminder hint switches its key and class on `remindInPast()`.
3. i18n keys in en and ar.

These are hints only: no validator, and saving is unchanged.

---

## Test Plan

Test projects are out of scope.

---

## Verification Steps

1. `npx ng build`: 0 errors, 0 warnings.
2. `/dashboard/tasks` → New task. On open, only the reminder hint shows.
3. Set Due to an hour ago and Reminder to two hours ago: both warnings show in the warning colour.
4. Set both in the future: back to the normal reminder hint and no due hint.

All passed on 2026-10-01.

---

## Done Criteria

- [x] A past due date shows the warning.
- [x] A past reminder shows the "sent right away" warning.
- [x] Future dates show the normal hints; en and ar keys exist.
- [x] `npx ng build` passes with 0 errors and 0 warnings.

**STOP HERE. Report to the user.**
