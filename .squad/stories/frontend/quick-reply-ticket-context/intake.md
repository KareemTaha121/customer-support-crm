# Story intake

- Folder: `.squad/stories/frontend/quick-reply-ticket-context/intake.md`

---

## Feature

- **Feature name (display):** Frontend (Agent Dashboard: quick replies)
- **Feature slug (folder under `plans/`):** `frontend`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-22`
- **Work item type:** `Bug`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `frontend`, `bug`, `quick-replies`

---

## Title

```
Quick replies fill customer and ticket placeholders in the ticket and chat composers
```

---

## Description

```
QA round 2 (N2). An agent saved the quick reply
  "Thanks {{customer.name}}, your ticket {{ticket.number}} is being handled by {{agent.name}}."
and picked it on a ticket page. The composer got
  "Thanks {{customer.name}}, your ticket {{ticket.number}} is being handled by QA Round2 Agent."

POST /quick-replies/{id}/render was sent without ?ticketId=. The picker has a ticketId
input (quick-reply-picker.component.ts:127) and passes it to the API, but neither
composer binds it:
- features/tickets/ticket-conversation.component.ts:103
- features/channels/chat-console.page.html:149
So only {{agent.name}} is ever filled, and an agent can send raw placeholders to a customer.
```

---

## Acceptance criteria

```
- [x] On ticket details the picker renders with ?ticketId=<ticket id>; all placeholders are filled.
- [x] In the live chat console the picker renders with the open conversation's ticket id.
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

- Repo root: `customer-support-crm-web/` (branch `main`). The render endpoint already checks the ticket is in scope (`AgentWorkspaceSlices.cs` 412–425).

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/ or e2e/.
