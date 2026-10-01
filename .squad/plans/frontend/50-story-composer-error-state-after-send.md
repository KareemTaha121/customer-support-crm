# Story 50 — Composers do not show "This field is required." after a successful send (Bug: BUG-14)

> Fix plan, not yet implemented. Paths and line numbers refer to customer-support-crm-web `06c817a` (`main`).
> Intake: [../../stories/frontend/composer-error-state-after-send/intake.md](../../stories/frontend/composer-error-state-after-send/intake.md)

## Prerequisites

- Story 12 — [12-story-channels-and-live-chat-console.md](12-story-channels-and-live-chat-console.md): the staff chat console.
- Story 17 — [17-story-customer-portal-ui.md](17-story-customer-portal-ui.md): portal ticket reply, portal live chat, portal chatbot.
- Source: manual QA report `.squad/qa/2026-10-01-manual-qa-report.md`, finding M1.
- All paths are relative to `customer-support-crm-web/src/app/` unless they start with `customer-support-crm-web/`.

---

## Story Goal

Four composers clear themselves with `FormGroup.reset()` after a successful send. `reset()` clears the value and the touched and dirty flags, but it does not touch the `FormGroupDirective`. Once the user has pressed the submit button, `FormGroupDirective.submitted` stays `true`. Angular Material's default `ErrorStateMatcher` puts a control in the error state when it is invalid and either touched or its form is submitted. The emptied field fails `Validators.required`, so it turns red at once.

| Composer | Before | After |
|---|---|---|
| Portal ticket reply | Red, "This field is required." after each send | Empty, neutral |
| Portal live chat | Red after a send, once the Send button has been used | Empty, neutral |
| Staff chat console | Red after a send; also red after switching conversation | Empty, neutral in both cases |
| Portal chatbot | Red outline and label, no text (no `<mat-error>`) | Empty, neutral |

Submitting an empty composer still shows the required error, and server field errors still show under the field.

**Why the staff ticket composer is not affected.** `features/tickets/ticket-conversation.component.ts` has no `<form>` and no `FormGroup`. The textarea is bound to a signal with `[ngModel]="body()" (ngModelChange)="body.set($event)"` (line 110) and has no `required` validator. The Send button is disabled until `canSend` (line 183) is true, and `send()` clears it with `this.body.set('')` (line 225). The control can never be invalid, so the error state never shows. That pattern is not reused here: it would drop the server-error display (`applyServerErrors`) that the four composers use.

**The pattern chosen.** The app already resets five forms through the directive (`resetForm()`, which also sets `submitted` back to `false`), always by passing a template reference into the submit handler, e.g. `features/auth/profile.page.ts:65` (`#formDirective="ngForm"`) and `:123` (`formDirective.resetForm()`). The broken composers also send from a keyboard handler and, in the console, reset from `open()`, where no template reference is passed. So this story keeps `resetForm()` but reads the directive with a signal query: `viewChild<FormGroupDirective>('<ref>')` on a named template reference. A name is needed because two of the pages hold two forms. No shared helper is added; each component gets one private `reset…()` method of three lines.

**Deviation from the intake:** none.

---

## Context — Read These Files First

1. `features/auth/profile.page.ts`: reference pattern, `#formDirective="ngForm"` (65), `changePassword(formDirective)` (113), `formDirective.resetForm()` (123).
2. `features/customer-portal/tickets/portal-ticket-detail.page.ts`: `replyForm` with `Validators.required` (80–82), `reply()` (138–162), the reset at 152.
3. `features/customer-portal/tickets/portal-ticket-detail.page.html`: the reply `<form>` (52) and its `<mat-error>` (57).
4. `features/customer-portal/channels/portal-chat.page.ts`: `messageForm` (97–99), `send()` (181–208), the reset at 195, `onComposerKey()` (210–215) which calls `send()` directly on Enter.
5. `features/customer-portal/channels/portal-chat.page.html`: the composer `<form>` (83) and `<mat-error>` (87).
6. `features/channels/chat-console.page.ts`: `composer` (115–117), `open()` with the reset at 195, `send()` (280–310) with the reset at 294, `onComposerKeydown()` (314–319, Ctrl/Cmd+Enter).
7. `features/channels/chat-console.page.html`: the composer `<form>` (135) inside `@if (current(); as conversation)` (69). Switching conversation keeps that branch rendered, so the same directive and its `submitted` flag survive.
8. `features/customer-portal/channels/portal-chatbot.page.ts`: `form` (69–71), `ask()` with the reset at 89, `reset()` (105–108) with the reset at 107, `onKey()` (110–115).
9. `features/customer-portal/channels/portal-chatbot.page.html`: the `<form>` (89); the field has no `<mat-error>` (90–93).
10. `features/tickets/ticket-conversation.component.ts`: the unaffected staff ticket composer (110, 183, 225).

### Sweep: every reset in `src/app`

Found with `grep -rn -E "\.reset\(|resetForm\(" src/app`.

| Location | Call | Affected | Why |
|---|---|---|---|
| `features/customer-portal/tickets/portal-ticket-detail.page.ts:152` | `replyForm.reset()` | **Yes** | Required `body`, form stays rendered |
| `features/customer-portal/tickets/portal-ticket-detail.page.ts:245` | `feedbackForm.reset({ comment })` | No | `comment` has only `maxLength` (85) |
| `features/customer-portal/channels/portal-chat.page.ts:139` | `startForm.controls.message.reset()` | No | Setting `stored` removes the start form (`@else if (!stored())`, html:4); "New chat" renders a fresh directive |
| `features/customer-portal/channels/portal-chat.page.ts:195` | `messageForm.reset()` | **Yes** | After any button send; the Enter path alone does not set `submitted` |
| `features/customer-portal/channels/portal-chatbot.page.ts:89` | `form.reset()` | **Yes** (outline only) | Same as above; no `<mat-error>` |
| `features/customer-portal/channels/portal-chatbot.page.ts:107` | `form.reset()` | **Yes** (outline only) | "New conversation" after a button send |
| `features/channels/chat-console.page.ts:195` | `composer.reset()` in `open()` | **Yes** | Directive survives a conversation switch |
| `features/channels/chat-console.page.ts:294` | `composer.reset()` | **Yes** | After any button send |
| `features/administration/audit/audit.page.ts:219` | `filters.reset()` | No | No validators (174) |
| `features/administration/organization/organization.page.ts:335, 342` | `profileForm.reset({...})`, `brandingForm.reset({...})` | No | Reset to the saved server values, which pass the validators (202–213) |
| `features/administration/users/user-dialog.component.ts:328` | `newPassword.reset()` | No | Standalone `FormControl` outside the `<form>` (form 77–96, input 149): no parent directive, and `reset()` clears touched |
| `features/customers/customer-notes.component.ts:218` | `editor.reset({...})` | No | The edit form is created per note inside `@if (editingId() === note.id)` (97–98) |
| `features/knowledge-base/staff/article-editor.page.ts:486, 506` | `form.reset({...})` | No | The form is removed while loading (80–85), and `new` / `articles/:id` are separate routes (`knowledge-base.routes.ts:21, 26`) |
| `features/tickets/ticket-list.page.ts:197` | `filters.reset()` | No | No validators (118–126) |
| `features/reports/report-filter-bar.component.ts:170` | `store.reset()` | No | Not a form |
| `features/auth/profile.page.ts:123` | `formDirective.resetForm()` | No | Already correct |
| `features/customer-portal/auth/portal-profile.page.ts:142` | `formDirective.resetForm()` | No | Already correct |
| `features/customer-portal/channels/portal-contact.page.ts:172` | `formDirective.resetForm()` | No | Already correct |
| `features/channels/channels-admin.page.ts:180` | `formDirective.resetForm({ channel, to: '' })` | No | Already correct |
| `features/customers/customer-notes.component.ts:205` | `directive.resetForm({ body: '', isPinned: false })` | No | Already correct |

Other forms that end in a successful submit either close their dialog or navigate away, so the directive is destroyed.

---

## Frontend Tasks

The same three steps in each affected component:

1. Add a template reference on the `<form>`: `#<name>="ngForm"` (the `ngForm` export of `FormGroupDirective`).
2. Read it: `private readonly <name> = viewChild<FormGroupDirective>('<name>');` and add `FormGroupDirective` (and `viewChild` where missing) to the imports.
3. Replace the `reset()` call with a private method that resets through the directive and falls back to the plain reset when the form is not rendered:

```ts
private resetReply(): void {
  const directive = this.replyFormRef();
  if (directive) {
    directive.resetForm();
  } else {
    this.replyForm.reset();
  }
}
```

`resetForm()` calls `FormGroup.reset()` and also sets `submitted` to `false`. The initial values are `''` (`NonNullableFormBuilder`), so the field ends up empty as before.

### 1 — Portal ticket reply

- `portal-ticket-detail.page.html:52`: `<form class="crm-card reply" #replyFormRef="ngForm" [formGroup]="replyForm" (ngSubmit)="reply()" novalidate>`.
- `portal-ticket-detail.page.ts`: import `viewChild` from `@angular/core` (line 1) and `FormGroupDirective` from `@angular/forms` (line 2). Add `private readonly replyFormRef = viewChild<FormGroupDirective>('replyFormRef');`. Line 152 becomes `this.resetReply();`.
- Leave `feedbackForm.reset(...)` (245) unchanged.

### 2 — Portal live chat

- `portal-chat.page.html:83`: `<form class="window__composer" #messageFormRef="ngForm" [formGroup]="messageForm" (ngSubmit)="send()" novalidate>`.
- `portal-chat.page.ts`: `viewChild` is already imported (line 11); add `FormGroupDirective` to line 13. Add `private readonly messageFormRef = viewChild<FormGroupDirective>('messageFormRef');`. Line 195 becomes `this.resetMessage();`.
- Leave line 139 unchanged (see the sweep).

### 3 — Staff chat console

- `chat-console.page.html:135`: `<form class="composer" #composerRef="ngForm" [formGroup]="composer" (ngSubmit)="send()" novalidate>`.
- `chat-console.page.ts`: `viewChild` is already imported (line 10); add `FormGroupDirective` to line 13. Add `private readonly composerRef = viewChild<FormGroupDirective>('composerRef');` next to `scroller` (88). Add `private resetComposer()` and call it at **both** 195 (`open()`) and 294 (`send()` success).
- In `open()` the directive may be absent (first conversation, or the previous one was closed and the form was not rendered); the fallback handles that.

### 4 — Portal chatbot

- `portal-chatbot.page.html:89`: `<form class="bot__composer" #composerRef="ngForm" [formGroup]="form" (ngSubmit)="ask()" novalidate>`.
- `portal-chatbot.page.ts`: `viewChild` is already imported (line 1); add `FormGroupDirective` to line 2. Add `private readonly composerRef = viewChild<FormGroupDirective>('composerRef');`. Lines 89 and 107 both become `this.resetComposer();`.
- `askSuggestion()` (94–97) calls `setValue` then `ask()`, so it is covered by line 89.

**No changes to:** `shared/**`, `core/**`, i18n files, the staff ticket composer, the forms marked "No" above, or the backend.

---

## Test Plan

Test projects are **out of scope**. The web repo has no tests for these components.

---

## Verification Steps

1. **Build:** `cd customer-support-crm-web && npx ng build` → 0 errors, 0 warnings.
2. **Portal ticket reply:** sign in to the portal, open a ticket, type a reply, click **Send reply** → the reply appears, the field is empty and not red. Click **Send reply** with the field empty → "This field is required." shows.
3. **Portal live chat:** start a chat. Send one message with the **Send** button and one with **Enter** → after each, the field is empty and not red. Press Send with an empty field → the required error shows.
4. **Staff chat console:** in `/chat`, open conversation A, send with the button and with Ctrl+Enter → the field is empty and not red. Click conversation B → its composer is empty and not red. Send an empty message → the required error shows.
5. **Portal chatbot:** ask a question with the button → the field is empty, outline not red. Click "New conversation" → same.
6. **Server error still shown:** in the portal ticket reply, type only spaces and click **Send reply**. `Validators.required` accepts it, `reply()` sends the trimmed empty body (line 144), and the API rejects it → the server message shows under the field (`showError` → `applyServerErrors`).
7. **Arabic:** repeat step 2 with the UI in Arabic.

---

## Done Criteria

- [ ] The portal ticket reply, portal live chat, staff chat console and portal chatbot are empty and neutral after a successful send, by button or keyboard.
- [ ] Switching conversation in the staff chat console leaves the composer neutral.
- [ ] An empty submit still shows the required error; server field errors still show.
- [ ] All four use the same `viewChild<FormGroupDirective>` + `resetForm()` pattern.
- [ ] Nothing changed in tests, e2e, `docker-compose.yml`, `deploy/` or `.github/`.
- [ ] `npx ng build` passes with 0 errors and 0 warnings.

**STOP HERE. Report to the user.**
