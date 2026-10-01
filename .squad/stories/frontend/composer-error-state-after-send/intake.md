# Story intake

- Folder: `.squad/stories/frontend/composer-error-state-after-send/intake.md`

---

## Feature

- **Feature name (display):** Frontend — Angular web app for all 12 features
- **Feature slug (folder under `plans/`):** `frontend`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-14`
- **Work item type:** `Bug`
- **Status:** `To do`
- **Assignee:** ``
- **Labels:** `frontend`, `bug`, `forms`, `portal`, `chat`

---

## Title

```
Composers do not show "This field is required." after a successful send
```

---

## Description

```
Manual QA 2026-10-01, finding M1 (.squad/qa/2026-10-01-manual-qa-report.md).
After a successful send, the emptied textarea immediately turns red and shows
"This field is required." in:

- the portal ticket reply   (features/customer-portal/tickets/portal-ticket-detail.page.ts:152)
- the portal live chat      (features/customer-portal/channels/portal-chat.page.ts:195)
- the staff chat console    (features/channels/chat-console.page.ts:195 and :294)
- the portal chatbot        (features/customer-portal/channels/portal-chatbot.page.ts:89, :107;
                             red outline only, it has no <mat-error>)

Cause: these handlers call FormGroup.reset(). That clears the value and the
touched flag but leaves FormGroupDirective.submitted === true after the user
pressed the submit button once. Angular Material's default ErrorStateMatcher
shows the error state when the control is invalid and (touched || form
submitted), and an empty required field is invalid.

The staff ticket composer is not affected: it has no form, no required
validator, and clears a signal (ticket-conversation.component.ts:110, :225).

Fix: reset through the FormGroupDirective (resetForm()), which also clears
`submitted`. The app already does this in five places (profile, portal profile,
portal contact, channel test message, customer notes). Use one pattern: a
template reference (#x="ngForm") read with viewChild(), so keyboard sends and
the console's conversation switch can reset it too.
```

---

## Acceptance criteria

```
- [ ] After a successful send, the four composers are empty and not red, whether sent by button or keyboard.
- [ ] Switching conversations in the staff chat console leaves the composer empty and not red.
- [ ] Submitting an empty composer still shows the required error.
- [ ] Server field errors after a failed send still show under the field.
- [ ] `npx ng build` passes with 0 errors and 0 warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** none.
- **Depends on code areas or other stories:** frontend stories 12 (chat console), 17 (customer portal).

## Technical hints (optional)

- Repo root: `customer-support-crm-web/` (branch `main`). Angular 22, Material 22, zoneless.
- Reference implementation: `features/auth/profile.page.ts:65, :113–123` (`formDirective.resetForm()`).

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify tests or e2e specs.
- A global custom `ErrorStateMatcher` (it would change every form in the app).
- The backend.
