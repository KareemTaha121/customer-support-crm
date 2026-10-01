# QA fix roadmap — manual QA of 2026-10-01 (BUG-11 to BUG-21)

> Source: [../qa/2026-10-01-manual-qa-report.md](../qa/2026-10-01-manual-qa-report.md). That was a manual browser pass (Chrome) over the staff app, customer portal, help center and API, on api `dca992c` (`develop`) and web `06c817a` (`main`).
> This file maps every finding to one story (intake + plan) and sets the execution order. Each plan is self-contained: give the implementing agent **only** that `NN-story-*.md` file.

## Finding → story

| Finding | Severity | NN | Id | Feature folder | Plan | Repos |
|---------|----------|----|----|----------------|------|-------|
| H1 No forgot/reset password (staff + portal) | High | 47 | BUG-11 | security-and-administration | [47](security-and-administration/47-story-password-reset.md) | api + web |
| H2 Virtual assistant shown when no AI provider is configured; no admin hint | High | 48 | BUG-12 | ai-features | [48](ai-features/48-story-hide-ai-when-unconfigured.md) | api + web |
| H3 Outbound email fails silently ("Reply sent." but outbox `Failed`) | High | 49 | BUG-13 | communication-channels | [49](communication-channels/49-story-outbound-delivery-visibility.md) | api + web |
| H4 Category cycles accepted | High | 43, 44 | BUG-07, BUG-08 | ticket-management, knowledge-base | [43](ticket-management/43-story-ticket-category-cycle-guard.md), [44](knowledge-base/44-story-knowledge-category-cycle-guard.md) | **Already done** (QA ran on a build from before `cedce62`); only re-verify |
| M1 "This field is required." after a successful send | Medium | 50 | BUG-14 | frontend | [50](frontend/50-story-composer-error-state-after-send.md) | web |
| M2 Category names stay English in the Arabic UI | Medium | 51 | BUG-15 | ticket-management | [51](ticket-management/51-story-localized-category-names.md) | api + web |
| M3 Bidi garbling of "name · date" in RTL + L5 mixed date formats | Medium / Low | 52 | BUG-16 | frontend | [52](frontend/52-story-rtl-bidi-isolation-and-date-formats.md) | web |
| M4 Live chat opening message invisible to agent and visitor | Medium | 53 | BUG-17 | communication-channels | [53](communication-channels/53-story-live-chat-opening-message.md) | api + web |
| M5 Audit log shows raw GUIDs and untranslated codes | Medium | 54 | BUG-18 | security-and-administration | [54](security-and-administration/54-story-readable-audit-log.md) | api + web |
| L1 "SLA breached" shown next to "in 15 hours"; breached counted as "at risk" | Low | 55 | BUG-19 | sla-and-automation | [55](sla-and-automation/55-story-sla-deadline-display.md) | api + web |
| L2 Portal nav scrollbar, L6 duplicate "Agent" label, form-field notch artifacts, page title | Low | 56 | BUG-20 | frontend | [56](frontend/56-story-ui-polish-qa-batch.md) | web |
| L3 Server-only validation in a snackbar, L4 Customer not marked required, L7 generic chatbot validation message, L8 English property names in Arabic | Low | 57 | BUG-21 | platform | [57](platform/57-story-validation-message-quality.md) | api + web |

## Execution order

Work one story at a time, in this order. Commit and push each repo after each story, and update this table, [00-index.md](00-index.md) and the feature `00-overview.md`.

| Wave | Stories | Why this order |
|------|---------|----------------|
| 0 | Restart the API on `develop` HEAD and re-check H4 (43/44) | QA ran on a build from before the fix |
| 1 | **49 ✅ → 47 ✅** | 47 sends reset links by email through the outbound pipeline; 49 adds the dev "log" email provider and delivery status that make 47 testable locally |
| 2 | **48** | Small and independent; removes a customer-facing internal error |
| 3 | **50, 56** | Frontend-only and low risk; 50 touches the composers that 53 also changes, so it goes first |
| 4 | **53** | Live chat transcript (after 50 to avoid conflicts in `chat-console.page.ts` / `portal-chat.page.ts`) |
| 5 | **51 → 52** | 51 adds Arabic category names to contracts; 52 then fixes bidi/date rendering on the same ticket templates |
| 6 | **54, 55** | Independent api+web stories |
| 7 | **57** | Last: it adds resx display names and validators across many slices, so rebasing it on everything else is simplest |

## Cross-story constraints

- **Migrations:** 47 (`AddPasswordReset`) and 54 (`AddAuditEntityLabel`) each add an EF migration. Generate each one on the latest `develop`, one story at a time, never in parallel.
- **Shared code:** 47, 51, 52, 55 and 56 touch `core/`, `shared/`, `styles.scss`, `app.config.ts` or `index.html`, which [frontend/00-overview.md](frontend/00-overview.md) reserves for platform stories. This is accepted for these cross-cutting bug fixes. Each plan names the shared files it edits.
- **Same files:** 50 → 53 (chat composers), 48 → 57 (`AiSlices.cs` chatbot handler/validator), 51 → 52 (ticket templates), 47 → 54 (54's i18n covers 47's new audit actions).

## Extra bugs found while planning (already inside the plans)

- **55:** the first-response "warned" timestamp is never cleared after the first reply (`Ticket.cs` 324–327), so answered tickets stay "At risk" until they are resolved.
- **49:** `POST /channels/outbox/{id}/retry` has no status check, so retrying a message that was already sent sends it again (adds 409 `OUTBOUND_ALREADY_SENT`).
- **48:** saving `/admin/settings` rebuilds the cached public flags from raw setting values, which turns the chatbot flag back on.
- **57:** customer contact email/phone format is checked only in the domain (first bad contact → 422 without a field), and `create()` drops empty contact rows, so server error indexes don't match form rows.

## Open decisions (defaults used by the plans; change before implementing if needed)

| Story | Question | Default in the plan |
|-------|----------|---------------------|
| 47 | Logging the Development reset link from the Application layer (first `ILogger` there) | Allowed; can move to Infrastructure |
| 49 | Where to show the failed-delivery banner | Channels page only (`channels.manage` users) |
| 49 | Notices with no ticket message (verification codes, "request received") | Visible only on the Channels page. Unconfigured email also blocks portal self-registration verification, which may need its own bug |
| 49 | Portal contact form text "We will reply by email." | Unchanged |
| 52 | `datetime-local` in the task dialog | Out of scope (follow-up with `mat-timepicker`) |
| 54 | Customer audit label | Number only (`C-000005`), so no personal data goes in the label |
| 54 | Position of the new CSV `entity_label` column | After `entity_id` (put it last if external tools parse the export) |
| 55 | Does a past late first response keep the ticket "breached" for life? | Yes (current meaning kept, with the text "First response was late") |
| 55 | Changing the meaning of `MyAtRisk` / `AtRiskTickets` to warned-only | Yes (only our own frontend reads them) |
| 57 | en/ar wording of the ~85 `Field_*` display names | Needs a native-speaker review; the keys are final |

## Rules (from `.squad/HANDOFF.md`)

- Verification is build-level: `dotnet build` with 0 warnings and `npx ng build` with 0 errors and 0 warnings, plus the manual steps in each plan.
- Do not touch Docker, `deploy/`, `.github/` or `tests/`.
- New user-visible text needs both `en` and `ar` keys (frontend i18n) or resx entries (backend).
- Conventional commits with the `Co-Authored-By` trailer.

## Round 2 findings

[../qa/2026-10-01-manual-qa-report-round2.md](../qa/2026-10-01-manual-qa-report-round2.md) re-verified 43–46 and found six new issues, N1–N6. N1 (portal self-registration blocked when email is unconfigured) was fixed in **49** (dev Log provider and a Settings warning). N2–N6 were fixed straight away as stories **58–62** (BUG-22 to BUG-26, as-built, see [00-index.md](00-index.md) "QA round 2 fixes"):

- N2: the quick reply picker never receives `[ticketId]`.
- N3: no warning for an Agent without a branch scope.
- N4: KB feedback has no limit.
- N5: `uploadedByName: null` in upload responses.
- N6: a task due date in the past is accepted.

## After all waves

Re-run the manual QA pass (same flows as the report) and append a "Re-test" section to the report.
