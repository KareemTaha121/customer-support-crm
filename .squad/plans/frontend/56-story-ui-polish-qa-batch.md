# Story 56 — UI polish batch from manual QA: portal nav scrollbar, duplicate "Agent" label, form-field row overlap, page title (Bug: BUG-20)

> Fix plan, not yet implemented. Paths and line numbers refer to customer-support-crm-web `06c817a` (`main`).
> Intake: [../../stories/frontend/ui-polish-qa-batch/intake.md](../../stories/frontend/ui-polish-qa-batch/intake.md)

## Prerequisites

- Story 08 — [08-story-core-platform-shell.md](08-story-core-platform-shell.md): `src/styles.scss`, `app.config.ts`, `BrandingService`, `index.html`.
- Story 10 — [10-story-customers-ui.md](10-story-customers-ui.md): the customer form.
- Story 11 — [11-story-tickets-ui.md](11-story-tickets-ui.md): the ticket conversation and the ticket create form.
- Story 17 — [17-story-customer-portal-ui.md](17-story-customer-portal-ui.md): the portal shell.
- Source: manual QA report `.squad/qa/2026-10-01-manual-qa-report.md`, findings L2 and L6. Issues 3 and 4 were noted in the same QA session but are not in the report table.
- All paths are relative to `customer-support-crm-web/src/app/` unless they start with `customer-support-crm-web/`.

**Ownership note.** This bug story edits shared files that feature stories may not touch (`src/styles.scss`, `core/branding/branding.service.ts`, `src/index.html`). That is intended: the causes live there.

---

## Story Goal

| # | Where | Before | After |
|---|---|---|---|
| 1 | Portal header (L2) | Vertical scrollbar (▲▼) next to "Help center" | No vertical scrollbar; horizontal scrolling on narrow screens still works |
| 2 | Staff ticket conversation (L6) | "Administrator  Agent  [Agent]" on staff replies of an agent-created ticket | "Administrator  Agent"; the channel pill shows only when it adds information (Email, WhatsApp, Portal, ...) |
| 3 | Customer form, ticket create form (and every other `crm-form-grid`) | Rows 4px apart; floated labels overlap the bottom border of the row above; short line segments between rows | 16px between rows; labels clear the row above |
| 4 | Browser tab title | "Customer Support CRM" during every load, then the organization name ("Customer Support" in the default seed) | The last known organization name from the start; "Customer Support CRM" only on a first visit |

### 1 — Why the portal nav scrolls vertically

`.portal-bar__nav` sets only `overflow-x: auto` (`features/customer-portal/portal-shell.component.ts:69`). When one axis is not `visible`, CSS computes the other one to `auto` too, so the nav is a scroll container on both axes.

The nav is a flex item of the toolbar row (`display: flex; align-items: center; height: 64px` in Material's `mat-toolbar-single-row`), so it is only as tall as its content: a `mat-button` is `height: 40px` (`--mat-button-text-container-height`). Each `mat-button` also contains `.mat-mdc-button-touch-target`, which is `position: absolute; top: 50%; height: 48px; transform: translateY(-50%)` (Material 22.2.1, `@angular/material/fesm2022/button.mjs`). It sticks out 4px above and 4px below the button. The 4px below counts as scrollable overflow, so `scrollHeight` is 44 against a `clientHeight` of 40, and the browser draws a vertical scrollbar.

**Deviation from the intake description:** the description said the mat-button is 44px. The button is 40px; the extra 4px is the 48px touch target.

### 2 — Where the second "Agent" comes from

The message header (`features/tickets/ticket-conversation.component.ts:67–76`) renders, in order: an icon (68), `authorName` (69), the **author type** as muted text (70, `tickets.authorType.Agent` = "Agent"), then either the "Internal note" pill (71–72) or a **channel** pill (73–74, `tickets.channel.<channel>`).

The pill is not the author type. It is the message channel. Staff replies take the ticket's channel (`customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Tickets/TicketMessageSlices.cs:99`, `ticket.Channel`), and a ticket created in the staff app defaults to `TicketChannel.Agent` (`.../Features/Tickets/TicketCommandSlices.cs:70`), whose English label is also "Agent" (`customer-support-crm-web/public/i18n/tickets/en.json:55`). `TicketMessage.channel` is a non-null `string` (`features/tickets/tickets.models.ts:100`), so the `@else if (message.channel)` branch always renders for public messages.

For other channels the pill is useful: on an Email ticket, a staff reply with "Email" tells the agent the reply went out by email. Only the "Agent" channel repeats what the author type says.

**Deviation from the intake description:** the task said "keep the chip". The chip is the channel, not the author type, so this plan keeps the author type text and hides the channel pill only for the "Agent" channel. Removing the author type text instead would leave customer messages on Email tickets with only an icon and "Email", and staff and customer messages on the same ticket would look alike.

### 3 — Why outlined fields overlap the row above

This is a real CSS issue, not a Material defect:

- The app sets `subscriptSizing: 'dynamic'` for every form field (`app.config.ts:25`). With `dynamic`, the subscript (hint or error line) takes no space unless it has content. With Material's default (`fixed`), every field reserves a ~20px line below itself, which is what normally separates rows.
- `.crm-form-grid` has `gap: 4px 16px` (`src/styles.scss:72`), so rows are 4px apart.
- In the outline appearance the floated label sits on the top border of the outline and its upper half is drawn above the field box (Material 22.2.1 moves it by `translateY(calc((6.75px + var(--mat-form-field-container-height, 56px) / 2) * -1))`, `@angular/material/fesm2022/_form-field-chunk.mjs`). With 4px between rows, that part of the label covers the bottom border of the field above.
- With the rows this close, the bottom border of one row and the top border of the next form two lines 4px apart. Where the next field opens its notch for the floated label, the upper line stays visible inside the gap. That is very likely the "short stray line segments" in the report.

The customer form's contact rows (`features/customers/customer-form.page.ts:230`, `.contact-row { gap: 4px 12px }`) have the same 4px row gap when a row wraps, and consecutive `.contact-row` blocks have no space between them at all.

The global grid is used by 17 files (customer form, ticket create, portal new ticket and contact forms, the admin, SLA, dashboard, KB and ticket category dialogs). They all get the larger gap, which is the intended effect.

**Rejected:** switching the global default back to `subscriptSizing: 'fixed'`. It would add ~20px under every field, including filter toolbars (`crm-toolbar`) and search boxes that do not set `subscriptSizing` themselves, and change the layout of every page.

**Not verified in a browser** while writing this plan (no dev server was running). The implementer must confirm the cause in DevTools before and after the change (Verification step 4). If line segments remain above an empty field once the rows are 16px apart, the cause is something else: inspect that field's `.mdc-notched-outline__leading`, `__notch` and `__trailing` elements and report back instead of adding more CSS.

### 4 — Why the title flips

`src/index.html:5` has `<title>Customer Support CRM</title>`. The app initializer (`app.config.ts:27–33`) awaits `BrandingService.load()`, and `apply()` calls `this.title.setTitle(branding.name)` (`core/branding/branding.service.ts:55–57`). The organization name comes from `GET /public/branding`; the seed default is "Customer Support" (`customer-support-crm-api/src/CustomerSupportCrm.Infrastructure/Persistence/Seed/BootstrapOptions.cs:18`, `customer-support-crm-api/src/CustomerSupportCrm.Api/appsettings.json:52`). So every full page load shows "Customer Support CRM" while the bundle loads and the branding call runs, then the organization name. The frontend fallback (`branding.service.ts:20–26`) is also "Customer Support CRM", used only when the branding call fails.

The organization name is editable (`features/administration/organization/organization.page.ts:346` re-applies it), so `index.html` cannot hard-code it. The fix remembers the last applied name and sets it before Angular boots.

---

## Context — Read These Files First

1. `features/customer-portal/portal-shell.component.ts`: the nav (28–37), its `mat-button` links (31), and the styles (63–75), with `.portal-bar__nav` at 69 and the mobile rule at 72–74.
2. `features/tickets/ticket-conversation.component.ts`: the message header (67–76). The author type text is at 70, the "Internal note" pill at 71–72, the channel pill at 73–74.
3. `features/tickets/tickets.models.ts:92–103`: `TicketMessage` (`channel: string`).
4. `customer-support-crm-web/public/i18n/tickets/en.json`: `channel.Agent` (55) and `authorType.Agent` (66).
5. `src/styles.scss`: `.crm-form-grid` (70–78), `gap: 4px 16px` at 72.
6. `app.config.ts:25`: `MAT_FORM_FIELD_DEFAULT_OPTIONS` with `subscriptSizing: 'dynamic'`.
7. `features/customers/customer-form.page.ts`: the profile grid (77), the contact rows (167–193), the styles (226–236) with `.contact-row` at 230.
8. `features/tickets/ticket-create.page.ts`: the grid (67–148), the styles (159–163).
9. `core/branding/branding.service.ts`: `FALLBACK` (20–26), `load()` (41–52), `apply()` (55–69).
10. `customer-support-crm-web/src/index.html:5`: the static title.
11. `core/localization/translation.service.ts:54–58` and `:120–128`: the existing `localStorage` pattern (`crm.language` key, `try`/`catch`).

---

## Frontend Tasks

### 1 — Portal nav: no vertical scrollbar

In `features/customer-portal/portal-shell.component.ts:69`, replace the rule:

```css
.portal-bar__nav { display: flex; align-self: stretch; align-items: center; gap: 4px; overflow-x: auto; overflow-y: hidden; }
```

- `align-self: stretch` makes the nav as tall as the toolbar row (64px, 56px on mobile), so the 48px touch targets fit inside it and keep their full hit area.
- `align-items: center` keeps the 40px buttons vertically centered in the taller nav.
- `overflow-y: hidden` removes the vertical scroll container even if a future density or theme change makes the content taller.
- Keep `overflow-x: auto`: on narrow screens the labels are hidden (72–74), but with many enabled items the nav must still scroll sideways.

### 2 — Ticket conversation: show "Agent" once

In `features/tickets/ticket-conversation.component.ts:73`, skip the channel pill for the "Agent" channel:

```html
@if (message.isInternal) {
  <span class="crm-pill crm-pill--warning">{{ 'tickets.reply.internalNote' | t }}</span>
} @else if (message.channel && message.channel !== 'Agent') {
  <span class="crm-pill">{{ 'tickets.channel.' + message.channel | t }}</span>
}
```

- Keep the author type text (70) and the "Internal note" pill unchanged.
- Do not change `tickets.channel.Agent` in the i18n files: the ticket list (`ticket-list.page.html:113`), the side panel (`ticket-side-panel.component.ts:159`) and the create form's channel picker (`ticket-create.page.ts:127`) use it for the ticket's channel.
- The portal conversation is not affected: `features/customer-portal/tickets/portal-ticket-detail.page.ts` does not render a channel.

### 3 — Form-field rows: room for the floated label

In `src/styles.scss:72`:

```scss
.crm-form-grid {
  display: grid;
  gap: 16px;
  grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
  ...
}
```

In `features/customers/customer-form.page.ts:230`:

```css
.contact-row { display: flex; flex-wrap: wrap; align-items: baseline; gap: 16px 12px; }
.contact-row + .contact-row { margin-block-start: 16px; }
```

- 16px is about the height of the part of the floated label that is drawn above the field, plus a few px of clear space. Use the same value in both places.
- Do not change `app.config.ts:25`.
- Do not add per-page overrides in the ticket create form: it uses `crm-form-grid` (67), so the global change covers it.

### 4 — Page title: the organization name from the start

In `core/branding/branding.service.ts`:

```ts
const TITLE_KEY = 'crm.brandName';

apply(branding: PublicBranding): void {
  this.branding.set(branding);
  this.title.setTitle(branding.name);
  if (branding !== FALLBACK) {
    try {
      localStorage.setItem(TITLE_KEY, branding.name);
    } catch {
      // ignore (private mode, storage disabled)
    }
  }
  ...
}
```

- `apply()` is called by `load()` and by the organization page after an admin saves (`organization.page.ts:346`), so a renamed organization is remembered too.
- The fallback is not stored, so a failed branding call does not overwrite the last real name.

In `customer-support-crm-web/src/index.html`, right after the `<title>` on line 5:

```html
<title>Customer Support CRM</title>
<script>try { var n = localStorage.getItem('crm.brandName'); if (n) { document.title = n; } } catch (e) {}</script>
```

- Keep "Customer Support CRM" as the first-visit default, so it matches `FALLBACK.name` and `core.appName` (`public/i18n/core/en.json:2`).
- No Content-Security-Policy is configured in the web repo, so the inline script runs. If a CSP is added later, this script needs its hash in `script-src`; add a comment with that note next to it.
- `help-article.page.ts:196` and `:200` keep setting "<article> · <organization>" and restoring the organization name; no change there.

**Simpler alternative, if an inline script is not wanted:** change `index.html:5` and `FALLBACK.name` to "Customer Support", the seed default. That removes the flip for the default organization only. Take the localStorage version unless the team decides against inline scripts.

**No changes to:** the backend, i18n files, `app.config.ts`, routes, or any other feature folder.

---

## Test Plan

Test projects are **out of scope**: do not create or modify unit or e2e tests.

---

## Verification Steps

1. **Build:** `cd customer-support-crm-web && npx ng build` → 0 errors, 0 warnings. Component styles must stay under the 8 kB budget.
2. **Portal nav (1):** open `/portal` signed out and signed in, at 1536px, 1280px and 375px wide (en and ar). No vertical scrollbar next to "Help center". In DevTools, `document.querySelector('.portal-bar__nav')` has `scrollHeight === clientHeight`. At 375px with all portal features enabled, the nav still scrolls horizontally if the icons do not fit.
3. **Conversation (2):** on a ticket created in the staff app, post a public reply → the header reads "<name>  Agent", with no "Agent" pill. Post an internal note → "Internal note" pill. On an Email or Portal ticket (or a ticket created by `/portal`), a staff reply still shows the "Email" or "Portal" pill, and the customer's message shows "Customer" and its channel pill. Repeat in Arabic.
4. **Form fields (3):** before the change, confirm the cause in DevTools on `/customers/new`: the floated label of a second-row field (e.g. "Preferred language", which has a default value) overlaps the bottom border of the field above it. After the change, on `/customers/new`, `/customers/{id}/edit`, `/tickets/new`, and one dialog (e.g. Tickets → Categories → New): floated labels clear the row above, no line segments show between rows, and an error message under a field does not collide with the next row. Add two contact rows in the customer form and check the space between them. Repeat in Arabic (RTL).
5. **Title (4):** with an empty `localStorage`, load `/login` → tab title "Customer Support CRM", then the organization name. Reload → the organization name from the start. Rename the organization in `/admin/organization` and reload → the new name from the start. Stop the API and reload → the last stored name stays (the fallback is not stored).

---

## Done Criteria

- [ ] The portal header nav has no vertical scrollbar at desktop and mobile widths, and still scrolls horizontally when needed.
- [ ] Staff replies on agent-created tickets show "Agent" once; other channel pills and the "Internal note" pill are unchanged.
- [ ] Rows in `crm-form-grid` forms and the customer contact rows are 16px apart; floated labels no longer overlap the row above.
- [ ] After the first visit, the tab title is the organization name from the start of a reload.
- [ ] Nothing changed in the backend, tests, `docker-compose.yml`, `deploy/` or `.github/`.
- [ ] `npx ng build` passes with 0 errors and 0 warnings.

**STOP HERE. Report to the user.**
