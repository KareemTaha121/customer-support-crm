# Story intake

- Folder: `.squad/stories/frontend/ui-polish-qa-batch/intake.md`

---

## Feature

- **Feature name (display):** Frontend — Angular web app for all 12 features
- **Feature slug (folder under `plans/`):** `frontend`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-20`
- **Work item type:** `Bug`
- **Status:** `To do`
- **Assignee:** ``
- **Labels:** `frontend`, `bug`, `ui`, `portal`, `tickets`, `forms`

---

## Title

```
UI polish batch from manual QA: portal nav scrollbar, duplicate "Agent" label, form-field row overlap, page title
```

---

## Description

```
Manual QA 2026-10-01 (.squad/qa/2026-10-01-manual-qa-report.md), findings L2 and
L6, plus two items noted during the same session (not in the report table).

1. Portal header (L2): a vertical scrollbar (up/down arrows) appears next to
   "Help center". features/customer-portal/portal-shell.component.ts:69 sets
   `.portal-bar__nav { overflow-x: auto }`, which makes overflow-y compute to
   auto. The nav is 40px high (mat-button height), but each mat-button has an
   absolutely positioned 48px touch target that sticks out 4px below it, so
   scrollHeight is 44 and the nav scrolls vertically. Measured at 1536px.

2. Staff ticket conversation (L6): a staff reply reads
   "Administrator  Agent  [Agent]". The muted text is the author type; the
   pill is the message CHANNEL (features/tickets/ticket-conversation.component.ts
   :70 and :74). Staff replies inherit the ticket's channel, and a ticket created
   in the staff app has channel "Agent", whose en label is also "Agent".

3. Outlined form fields in the customer form and the ticket create form: rows
   sit 4px apart (`.crm-form-grid` row gap, src/styles.scss:72) and the global
   default `subscriptSizing: 'dynamic'` (app.config.ts:25) reserves no space
   under a field. A floated outline label sits on the top border and its upper
   half is drawn above the field box, over the bottom border of the row above.
   Short line segments show between rows.

4. The browser tab title is "Customer Support CRM" (index.html:5) until the app
   initializer applies the organization name from GET /public/branding
   ("Customer Support" in the default seed), on every full page load.

Fix (frontend only):
- 1: `.portal-bar__nav` gets `align-self: stretch; align-items: center;
  overflow-y: hidden;`.
- 2: hide the channel pill when the channel is "Agent"; keep the author type
  text, the "Internal note" pill and the other channel pills.
- 3: increase the `.crm-form-grid` row gap to 16px; the same for the
  customer form's contact rows.
- 4: remember the last applied organization name in localStorage and set
  document.title from it in index.html before Angular boots.
```

---

## Acceptance criteria

```
- [ ] The portal header nav shows no vertical scrollbar at 1536px, 1280px and 375px; it still scrolls horizontally when the items do not fit.
- [ ] A staff reply on an agent-created ticket shows the author type once; replies on Email, WhatsApp, Portal, etc. tickets still show the channel pill; internal notes still show "Internal note".
- [ ] In the customer form and the ticket create form, floated labels do not touch the row above and no stray line segments show between rows (en and ar).
- [ ] After the first visit, a full page reload keeps the organization name as the tab title from the start.
- [ ] `npx ng build` passes with 0 errors and 0 warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** none.
- **Depends on code areas or other stories:** frontend stories 08 (shell, global styles, branding), 10 (customer form), 11 (tickets), 17 (customer portal).

## Technical hints (optional)

- Repo root: `customer-support-crm-web/` (branch `main`). Angular 22, Material 22 (22.2.1), zoneless.
- Staff replies use the ticket's channel: `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Tickets/TicketMessageSlices.cs:99`.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify tests or e2e specs.
- The backend.
- Covered by other stories: composer error after send (50), category names (51), bidi and date formats (52), SLA display (55), validation and the required marker (57).
