# Story intake

- Folder: `.squad/stories/frontend/rtl-bidi-isolation/intake.md`

---

## Feature

- **Feature name (display):** Frontend — Angular web app for all 12 features
- **Feature slug (folder under `plans/`):** `frontend`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-16`
- **Work item type:** `Bug`
- **Status:** `To do`
- **Assignee:** ``
- **Labels:** `frontend`, `bug`, `i18n`, `rtl`, `dates`

---

## Title

```
Isolate mixed-direction values in RTL and use one date format rule
```

---

## Description

```
Manual QA 2026-10-01 (.squad/qa/2026-10-01-manual-qa-report.md), findings M3 and L5.

M3 — bidi reordering in Arabic. The ticket history meta line
  {{ entry.actorName ?? ... }} · {{ entry.occurredAt | localDate }}
(features/tickets/ticket-history.component.ts:45) should read
"Administrator · 01/10/2026، 6:53 ص" but renders scrambled
("01 · 2026/10/Administrator..."). The Arabic date from Intl
(ar-SA-u-nu-latn-ca-gregory) contains U+200F RIGHT-TO-LEFT MARK after the day
and the month ("01‏/10‏/2026، 6:53 ص"); next to a Latin name and a
neutral " · " in an RTL paragraph, the Unicode bidi algorithm reorders the
runs. The same "value · value" pattern appears in about 20 templates.

Fix:
- The localDate pipe wraps its output in FSI…PDI (U+2068…U+2069), so every
  date is isolated, including dates inside translated sentences.
- The t pipe wraps string parameters the same way ({{name}}, {{number}}…).
- Data values (names, numbers, emails, codes, categories) in "a · b" meta
  lines are wrapped in <bdi>.

L5 — mixed date formats. One ticket page shows "1 Oct 2026, 06:53" (medium),
"01/10/2026, 06:53" (short) and "14 seconds ago" (relative). Rule:
- absolute date + time: 'medium'; date only: 'mediumDate';
- 'shortTime' only inside chat bubbles and "last updated";
- 'relative' only in lists, queues, feeds and conversation threads, always
  with the absolute value as a tooltip, or as a "(in 15 hours)" supplement
  after an absolute due date.
The 'short' and 'shortDate' formats are removed from the pipe.

Date inputs. The five <input type="date"> fields show the browser's
mm/dd/yyyy while the app shows dd/mm/yyyy. Replace them with the Material
datepicker on one app-wide, language-aware DateAdapter that also parses typed
dd/mm/yyyy.
```

---

## Acceptance criteria

```
- [ ] In Arabic, the ticket history line reads "Administrator · <date>" with the date intact.
- [ ] Every "value · value" meta line listed in the plan keeps its order in Arabic.
- [ ] Dates inside translated sentences (SLA due, task due, "updated") are intact in Arabic.
- [ ] Absolute dates use 'medium' / 'mediumDate' everywhere; relative times have an absolute tooltip.
- [ ] Date filters (tickets, audit) and the API key expiry use the Material datepicker, show dd/mm/yyyy, and accept typed dd/mm/yyyy.
- [ ] `npx ng build` passes with 0 errors and 0 warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** none. Related: M2 (Arabic category names) is a separate story.
- **Depends on code areas or other stories:** frontend story 08 (`localDate` pipe, `TranslatePipe`), 11 (tickets), 18 (reports datepicker).

## Technical hints (optional)

- Repo root: `customer-support-crm-web/` (branch `main`). Angular 22, Material 22.
- Existing isolation examples: `channels-admin.page.html:104` (`<bdi>`), `sla-policies.page.ts:97` (`<bdi>`), `settings.page.ts:57` (`dir="ltr"`).

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify tests or e2e specs.
- The `datetime-local` inputs in the task dialog (`dashboard/task-dialog.component.ts:67, 72`); see the plan's open question.
- Translating the datepicker's own labels (`MatDatepickerIntl`).
- Arabic-Indic digits; the app keeps Latin digits in both languages.
- The backend.
