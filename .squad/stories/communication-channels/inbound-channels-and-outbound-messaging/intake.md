# Story intake

- Folder: `.squad/stories/communication-channels/inbound-channels-and-outbound-messaging/intake.md`

---

## Feature

- **Feature name (display):** Communication Channels (feature 03)
- **Feature slug (folder under `plans/`):** `communication-channels`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `CH-01`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `communication-channels`

---

## Title

```
Inbound channels (email / WhatsApp / SMS / web form) and outbound customer messaging
```

---

## Description

```
As built in customer-support-crm-api e921626 (+ c8e9251). Code in
Application/Features/Channels/{InboundChannels.cs, CustomerMessaging.cs},
Domain/Tickets/ChannelRecords.cs, Infrastructure/Channels/*.

Inbound (anonymous, /api/v1/public):
- GET  /channels/{channel}/webhook   provider handshake (WhatsApp hub.verify_token); 403 otherwise
- POST /channels/{channel}/webhook   email | whatsapp | sms (case-insensitive)
    email:    X-Webhook-Secret header == Channels:Email:InboundSecret, JSON {messageId, from, fromName, to, subject, text}
    whatsapp: X-Hub-Signature-256 HMAC-SHA256 (Channels:WhatsApp:AppSecret), Cloud API payload
    sms:      X-Twilio-Signature HMAC-SHA1 (Channels:Sms:AuthToken + InboundWebhookUrl), form post
  Unknown channel -> 404; failed verification -> 401; success -> 200 (no envelope).
- POST /web-forms/tickets            {name, email, phone?, subject, message, categoryId?, language?, website}
                                     rate limit "public"; toggle webform.enabled; 201 {ticketNumber}
- GET  /web-forms/categories         active ticket categories (c8e9251); rate limit "public"

Inbound pipeline (ProcessInboundMessageCommand):
- Dedupe by (channel, externalId) in inbound_receipts (unique index).
- Customer matched by normalized email / phone (phone and WhatsApp contacts are interchangeable);
  unknown senders always get a new Individual customer tagged "auto-created".
- Email routed to the department whose Email equals the "to" address.
- Email threads by "[T-000123]" subject tag; WhatsApp/SMS continue the customer's latest
  non-closed ticket on the same channel updated within 7 days; otherwise a new ticket.

Outbound (transactional outbox, outbound_messages):
- Agent replies, "received" and "resolved" notices queued by domain-event handlers;
  WhatsApp/SMS tickets get replies on that channel, everything else by branded en/ar email.
- DispatchOutboxCommand every 15 s, batch 50, exponential backoff 1/2/4/8/16 min, max 6 attempts;
  4xx (except 429) and unconfigured channels fail permanently.
- Providers: SMTP (System.Net.Mail), WhatsApp Cloud API, Twilio-compatible SMS.

Administration (/api/v1, channels.manage):
- GET  /channels/status              configured flag per sender (from configuration)
- POST /channels/test                {channel: Email|WhatsApp|Sms, to} queues a test message
- GET  /channels/outbox              ?status=Pending|Sent|Failed&page=  (50 per page)
- POST /channels/outbox/{id}/retry   404 OUTBOUND_MESSAGE_NOT_FOUND

Errors: VALIDATION_ERROR (INVALID honeypot, INVALID_VALUE language/channel), FEATURE_DISABLED 409,
NO_ACTIVE_BRANCH 404, OUTBOUND_MESSAGE_NOT_FOUND 404.
Channel credentials come from configuration (Channels:Email|WhatsApp|Sms); appsettings.json ships empty values.
```

---

## Acceptance criteria

```
- [ ] Provider-neutral abstractions: ICommunicationChannel, IMessageSender, IMessageReceiver.
- [ ] Provider SDKs only in Infrastructure; no per-provider ticket business logic.
- [ ] Inbound messages normalized to a common InboundMessage model and matched to customer (by email/phone) and ticket (by thread/reference).
- [ ] Unknown senders create a new customer (or lead) record per configuration.
- [ ] Webhooks validated (signatures), idempotent (deduplicate by provider message id).
- [ ] Outbound sending runs via background jobs with retries and delivery status tracking.
- [ ] Web forms protected against spam/abuse (rate limiting).
- [ ] Channel credentials stored as secrets, never in source.
- [ ] Admin can configure channel accounts (mailboxes, numbers, widget) per branch/department.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** customer-management (NN 23–24), ticket-management (NN 25–26), platform (NN 20–21: background jobs, realtime, settings).
- **Depends on code areas or other stories:** `TicketFactory`, `TicketMessageWriter`, `TicketQueries`, ticket domain events, `Customer` / `CustomerContact`, `ISequenceGenerator`, `AddRecurringRequest`, `RateLimitPolicies.Public`, `IPublicEndpoint`.

## Extra notes (optional)

- Live chat is a separate story: CH-02 (`live-chat`).
- `CustomerMessenger.QueueVerificationCodeAsync` is used by the customer portal (feature 08).
- Frontend: `../../../plans/frontend/12-story-channels-and-live-chat-console.md` (FE-05).

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- IMAP mailbox polling, WhatsApp approved templates, provider delivery-receipt callbacks.
