# Story intake

- Folder: `.squad/stories/communication-channels/outbound-delivery-visibility/intake.md`

---

## Feature

- **Feature name (display):** Communication Channels
- **Feature slug (folder under `plans/`):** `communication-channels`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-13`
- **Work item type:** `Bug`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `frontend`, `bug`, `channels`, `tickets`

---

## Title

```
Show staff when a reply to the customer was not delivered
```

---

## Description

```
Manual QA 2026-10-01, finding H3 (.squad/qa/2026-10-01-manual-qa-report.md).

With the Email channel not configured, a staff public reply shows "Reply sent."
and the portal contact form says "We will reply by email.". GET /channels/outbox
then holds the messages as status Failed, attempts 1,
lastError "The Email channel is not configured.". Nothing on the ticket tells
staff that the customer never got the reply.

The data already exists: every agent reply queued by CustomerMessenger carries
TicketId and TicketMessageId on its OutboundMessage row. Only the Channels admin
page (channels.manage) shows it.

Fix:
1. GET /tickets/{id}/messages (and the POST reply response) return a
   per-message delivery block for agent replies: outbox id, channel, status,
   attempts, sentAt, whether the channel is configured, and lastError for
   channels.manage holders only.
2. The staff ticket conversation shows a small chip on such messages
   (Queued / Sent / Not delivered), with a Retry button for channels.manage
   (existing POST /channels/outbox/{id}/retry). The reply toast warns when the
   reply channel is not configured, instead of "Reply sent.".
3. The Channels page shows a banner with the number of failed messages and a
   "Show failed" shortcut.
4. Retry refuses messages that were already sent (today it re-sends them).
5. A Development-only "Log" email provider (Channels:Email:Provider = "Log")
   writes outgoing email to the log and marks it sent. It is off by default and
   reports "not configured" outside Development.
```

---

## Acceptance criteria

```
- [ ] GET /tickets/{id}/messages returns delivery { outboundMessageId, channel, status, attempts, sentAt, channelConfigured, lastError } on agent replies that were queued, and null otherwise.
- [ ] lastError is null for callers without channels.manage.
- [ ] Portal and live chat message lists never include delivery data.
- [ ] The staff conversation shows Queued / Sent / Not delivered chips (en/ar); channels.manage users can retry a failed delivery from the ticket.
- [ ] A public reply on a ticket whose reply channel is not configured shows a warning toast instead of "Reply sent.".
- [ ] The Channels page shows the failed-message count banner with a "Show failed" shortcut.
- [ ] POST /channels/outbox/{id}/retry on a sent message returns 409 OUTBOUND_ALREADY_SENT (en/ar).
- [ ] With Channels:Email:Provider = "Log" in Development, replies are marked Sent and logged; in other environments the Email channel reports "not configured".
- [ ] `dotnet build` passes with zero warnings; `ng build` passes with zero errors and zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** none.
- **Depends on code areas or other stories:** story 32 (inbound channels and outbound messaging: outbox, `CustomerMessenger`, Channels admin endpoints), frontend stories 11 (ticket conversation) and 12 (Channels page).

## Technical hints (optional)

- API repo: `customer-support-crm-api/` (branch `develop`). .NET 10.
- Web repo: `customer-support-crm-web/` (branch `main`). Angular.
- `OutboundMessage.TicketMessageId` already exists and is set for agent replies; no migration is needed.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- No e2e tests.
- Ticket-level notices ("request received", "request resolved") and portal verification codes: they have no ticket message and stay visible on the Channels page only.
- Changing the portal contact form wording, or the retry/backoff schedule.
- A staff-shell or dashboard-wide banner; realtime push of delivery status changes.
- A real SMTP provider, or any production configuration.
