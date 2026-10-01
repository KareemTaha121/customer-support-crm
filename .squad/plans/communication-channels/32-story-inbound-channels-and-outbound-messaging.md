# Story 32 — Inbound channels and outbound customer messaging (Story: CH-01)

> As-built plan: written after implementation in `customer-support-crm-api` commit `e921626` (feat: add communication channels) with follow-up `c8e9251` (public web-form categories). Paths and line numbers refer to `c8e9251`; every cited file is unchanged at `develop` HEAD `2956767` except where another commit is named.

## Prerequisites

- platform (NN 20–21, `65c74a3`): `IPublicEndpoint` group `/api/v1/public`, `AddRecurringRequest`, `IRealtimeNotifier`, `ISequenceGenerator`, `IAccessScopeProvider`.
- customer-management (NN 23–24, `0fd694e`): `Customer`, `CustomerContact.NormalizeValue`, `ContactType`, `PhoneNumber`.
- ticket-management (NN 25–26, `57e52f8`): `TicketFactory`, `NewTicket`, `TicketMessageWriter`, `TicketQueries`, `TicketChannel` (`Email`, `WhatsApp`, `Sms`, `WebForm`, `Chat`, …), `TicketMessage.Channel` / `ExternalMessageId`, `Ticket.ReplyAddress`, domain events `TicketCreatedDomainEvent`, `TicketMessageAddedDomainEvent`, `TicketStatusChangedDomainEvent`.
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

1. Accept signed provider webhooks for email, WhatsApp and SMS, normalize them into one `InboundChannelMessage`, dedupe, match/create the customer and either open a ticket or thread a reply.
2. Accept an anonymous web form that creates a ticket (honeypot + rate limit) and list categories for it.
3. Deliver customer-facing messages (agent replies, "request received", "request resolved", portal verification codes) through a transactional outbox with retries.
4. Give administrators channel status, a test send, an outbox view and manual retry.

**Deviations from the intake (the code is authoritative):**

| Feature spec (`03-communication-channels.md`) | As built in `e921626` |
|---|---|
| `ICommunicationChannel`, `IMessageSender`, `IMessageReceiver` | `IMessageSender` (Application abstraction) + `IChannelWebhookAdapter` (declared in `Features/Channels/InboundChannels.cs`, not under `Abstractions/`); no `ICommunicationChannel` / `IMessageReceiver` |
| Common `InboundMessage` model | `InboundChannelMessage` record |
| `ChannelConfiguration` entity, Configurations CRUD per branch/department | **Not built.** Providers are configured only through options (`Channels:Email`, `Channels:WhatsApp`, `Channels:Sms`); one account per channel for the whole organization. `GET /channels/status` reports `IsConfigured` |
| Test connection | `POST /channels/test` queues a test message in the outbox (no synchronous provider check) |
| Mailbox polling (IMAP) | Not built; email inbound is a generic JSON webhook guarded by a shared secret header |
| WhatsApp approved templates; reply templates | Not built; WhatsApp sends free text (24-hour window only). Email notices use built-in en/ar templates (`CustomerTemplates`) |
| `TicketMessage.DeliveryStatus`, `Direction`, `ProviderMessageId` | Delivery status lives on `OutboundMessage` (`Pending`/`Sent`/`Failed`, `ProviderMessageId`, `LastError`), linked by `TicketMessageId`; no provider delivery-receipt callbacks (`Sent` = accepted by provider) |
| Unknown senders create customer or lead "per configuration" | Always creates an `Individual` customer tagged `auto-created`; no setting |
| Slices `Inbound/ReceiveEmail` … `Outbound/SendReply, TrackDelivery` | Grouped files: `InboundChannels.cs`, `CustomerMessaging.cs`; agent replies reuse the ticket reply endpoint and are picked up by event handlers |
| Webhooks validated by signature | Email: shared secret header (not a signature); WhatsApp: HMAC-SHA256; SMS: Twilio HMAC-SHA1 |
| Migration with the feature | No migration in `e921626`; tables `outbound_messages`, `inbound_receipts`, `chat_conversations` were created by `20260930104602_AddSupportOperations` in `678ea67` |
| — | Follow-up `678ea67`: `POST /public/web-forms/tickets` gated by setting `webform.enabled` (`FeatureToggleBehavior`) |
| — | Follow-up `c8e9251`: `GET /public/web-forms/categories` |

**Not in scope:** live chat (CH-02). **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Api/Endpoints/EndpointExtensions.cs` (`65c74a3`) — lines 16–46; `/api/v1/public` group is `AllowAnonymous()` (30–34), staff group requires `PolicyNames.Staff` (18).
2. `src/CustomerSupportCrm.Application/Abstractions/Channels/ChannelAbstractions.cs` — `OutboundEnvelope` (5), `SendResult` with `Permanent` (8–13), `IMessageSender` (16–23), `InboundChannelMessage` (29–37), `CustomerPortalOptions` section `Portal` (40–46).
3. `src/CustomerSupportCrm.Domain/Tickets/ChannelRecords.cs` — `OutboundStatus` (7–12), `OutboundMessage` (18–116: `MaxAttempts = 6` line 20, `Queue` 63–86 only Email/WhatsApp/Sms, `MarkSent` 88–95, `MarkFailed` 97–109 backoff `2^(attempts-1)` minutes, `Retry` 111–115), `InboundReceipt` (118–142).
4. `src/CustomerSupportCrm.Application/Abstractions/Persistence/IApplicationDbContext.Channels.cs` lines 6–13 and `src/CustomerSupportCrm.Infrastructure/Persistence/ApplicationDbContext.Channels.cs` lines 6–13 — `OutboundMessages`, `InboundReceipts`, `ChatConversations`.
5. `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/ChannelConfiguration.cs` — `OutboundMessageConfiguration` (8–24, index `(Status, NextAttemptAt)`, `TicketId`), `InboundReceiptConfiguration` (26–37, **unique** `(Channel, ExternalId)`).
6. `src/CustomerSupportCrm.Infrastructure/Persistence/Migrations/20260930104602_AddSupportOperations.cs` (`678ea67`) — `inbound_receipts` 165–179, `outbound_messages` 257–280, indexes 1112–1116 and 1149–1157.
7. `src/CustomerSupportCrm.Application/Features/Channels/InboundChannels.cs` — `IChannelWebhookAdapter` (21–34), `CustomerResolver` (36–78), `ProcessInboundMessageHandler` (80–168), `ChannelWebhookEndpoints` (170–211), web form (213–262), `WebFormEndpoints` (264–284).
8. `src/CustomerSupportCrm.Application/Features/Channels/CustomerMessaging.cs` — `CustomerTemplates` (22–61), `CustomerMessenger` (63–161), `ChannelDeliveryHandlers` (165–229), `DispatchOutboxHandler` (233–283), administration (285–362).
9. `src/CustomerSupportCrm.Infrastructure/Channels/ChannelOptions.cs` — `EmailOptions` (4–26, `InboundSecret` 25), `WhatsAppOptions` (29–46), `SmsOptions` (49–65, `InboundWebhookUrl` 64).
10. `src/CustomerSupportCrm.Infrastructure/Channels/MessageSenders.cs` — `SmtpEmailSender` (17–61), `WhatsAppSender` (64–106), `TwilioSmsSender` (109–143).
11. `src/CustomerSupportCrm.Infrastructure/Channels/WebhookAdapters.cs` — `EmailWebhookAdapter` (18–56), `WhatsAppWebhookAdapter` (59–128), `SmsWebhookAdapter` (131–172).
12. `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` at `e921626` — `services.AddChannels()` (47), `AddChannels` (96–112, recurring dispatch line 111). `src/CustomerSupportCrm.Application/DependencyInjection.cs` at `e921626` — `CustomerMessenger`, `CustomerResolver` (45–46).
13. `src/CustomerSupportCrm.Application/Features/Tickets/TicketCommandSlices.cs` — `TicketFactory.CreateAsync` (from line 107) reads `db.Customers.Local` first (lines 111–117, `CustomerDefaults` record 162; changed in `e921626`) so a customer created in the same unit of work can receive a ticket.
14. `src/CustomerSupportCrm.Domain/Roles/Permissions.cs` (`65c74a3`) — `ChannelsManage = "channels.manage"` (32); not in `AgentDefaults` / `ManagerDefaults` (administrators only).
15. `src/CustomerSupportCrm.Application/Abstractions/Http/RateLimitPolicies.cs` line 9 (`Public`); policy in `src/CustomerSupportCrm.Api/Configuration/SecurityExtensions.cs` lines 141–152 at `2956767` (per-IP fixed window, `RateLimiting:Public` 30 / 60 s in `appsettings.json` lines 36–39).
16. `src/CustomerSupportCrm.Application/Features/Settings/SettingsSlices.cs` (`678ea67`) — `FeatureToggleBehavior` 135–160 (`SubmitWebFormCommand` → `webform.enabled`, line 143); `EnsureEnabledAsync` 49–55 throws `ConflictException(FEATURE_DISABLED)`.
17. `docs/endpoints.md` — Channels 148–153, Public 196–211.

---

## Backend Tasks

### 1 — Domain and persistence

`OutboundMessage` is a transactional outbox row written in the same `SaveChanges` as the change that caused it. `InboundReceipt` records each processed provider message id. EF configurations map enums as strings (max 20) and add the indexes in Context 5. DbSets are partial-interface/partial-class additions (Context 4).

### 2 — Abstractions and providers

- `IMessageSender { TicketChannel Channel; bool IsConfigured; Task<SendResult> SendAsync(OutboundEnvelope, ct) }`.
- `SmtpEmailSender` — multipart text + HTML via `System.Net.Mail.SmtpClient`; invalid recipient → permanent failure. `IsConfigured` = `Enabled && Host && FromAddress`.
- `WhatsAppSender` — `POST {ApiBaseUrl}/{PhoneNumberId}/messages` (Graph v20.0), bearer token, text message; returns `messages[0].id`. 4xx except 429 → permanent.
- `TwilioSmsSender` — `POST {ApiBaseUrl}/2010-04-01/Accounts/{AccountSid}/Messages.json`, basic auth, body cut at 1600 chars; returns `sid`.
- DI (`AddChannels`): options bound to `Channels:Email|WhatsApp|Sms` and `Portal`; `SmtpEmailSender` singleton; the two HTTP senders via `AddHttpClient<IMessageSender, …>` with 20 s timeout; three adapters as singletons; `AddRecurringRequest<DispatchOutboxCommand>(15 s)`.

### 3 — Webhooks

`ChannelWebhookEndpoints : IPublicEndpoint` maps `/channels/{channel}/webhook` (→ `/api/v1/public/channels/{channel}/webhook`):

- `GET` → `adapter.Handshake(request)`; returns the challenge as text, else `Results.Forbid()`. Only WhatsApp implements it (`hub.mode=subscribe`, constant-time compare of `hub.verify_token`).
- `POST` → unknown channel 404; `ParseAsync` returning `null` (verification failed or secret not configured) → 401; each parsed message sent as `ProcessInboundMessageCommand`; 200. `.DisableAntiforgery()`.
- Email adapter: `X-Webhook-Secret` vs `InboundSecret` (`FixedEquals`, constant time); body `{ messageId, from, fromName, to, subject, text }`; `"Name <a@b>"` → address. Missing `from`/`messageId` → empty list (200, nothing processed).
- WhatsApp adapter: buffers the body, verifies `X-Hub-Signature-256 = "sha256=" + hex(HMACSHA256(AppSecret, body))`; walks `entry[].changes[].value.messages[]`; non-text messages become `"[<type> message]"`; sender `"+" + from`; profile name from `contacts[]`.
- SMS adapter: requires form content, `AuthToken`, `InboundWebhookUrl`; signature base = URL + sorted `key+value` pairs, HMAC-SHA1, base64 (`#pragma warning disable CA5350` with justification).

### 4 — Inbound pipeline

`ProcessInboundMessageCommand(InboundChannelMessage) : IRequest<Guid?>` handled by `ProcessInboundMessageHandler` (97–143):

1. `InboundReceipts.AnyAsync(channel, externalId)` → return `null` (duplicate, lines 100–103).
2. Contact type: Email / WhatsApp / otherwise Phone.
3. Email only: department whose `Email == EmailAddress.Normalize(to)` and active (112–117).
4. `CustomerResolver.ResolveAsync` — match on normalized contact value; phone and WhatsApp contacts match each other (42); otherwise create `Customer` with next `Sequences.CustomerNumbers`, `CustomerType.Individual`, name or address, language `en`, tag `auto-created`, branch/department from `DefaultUnitAsync` (routed department, else oldest active branch; none → `NotFoundException(NO_ACTIVE_BRANCH)`).
5. `FindThreadAsync` (145–164): email by `\[(T-\d{6,})\]` in subject **and** same customer; WhatsApp/SMS latest non-closed ticket of the customer on that channel with `UpdatedAt ?? CreatedAt` within 7 days.
6. No thread or closed ticket → `TicketFactory.CreateAsync(new NewTicket(..., TicketPriority.Medium, channel, branch, department, [], replyAddress))`, subject = provider subject or first 80 chars of body. Otherwise `TicketMessageWriter.AddAsync(... MessageAuthorType.Customer ..., channel, externalId ...)`.
7. Add `InboundReceipt`, `SaveChangesAsync`, return ticket id.

### 5 — Web form

- `WebFormTicketRequest(Name, Email, Phone?, Subject, Message, CategoryId?, Language?, Website?)`; `Website` is a honeypot.
- `SubmitWebFormValidator` (222–234): name ≤ `Customer.NameMaxLength`, email valid ≤ 254, phone ≤ 32, subject ≤ `Ticket.SubjectMaxLength`, message ≤ 10 000, `Website` empty (`INVALID`), language `null|en|ar` (`INVALID_VALUE`).
- `SubmitWebFormHandler` (240–262): resolve customer by email (nothing about an existing record is returned); adds a valid phone as non-primary contact; ignores an unknown/inactive category; creates a `WebForm` ticket with reply address = normalized email; returns `WebFormTicketResponse(TicketNumber)`.
- Endpoints (264–284): `POST /web-forms/tickets` → 201, `GET /web-forms/categories` → `PortalCategoryResponse(Id, Name, NameAr)` ordered by `SortOrder`, `Name`; both `.RequireRateLimiting(RateLimitPolicies.Public)`.

### 6 — Outbound messaging

- `CustomerMessenger` (scoped): `QueueAgentReplyAsync` — WhatsApp/SMS with `ReplyAddress` → same channel, raw body; `Chat` → nothing (SignalR); else email to `ReplyAddress ?? PrimaryEmail` with subject `"[T-000123] subject"` (`EmailSubject`, used for threading) and a portal link `{Portal:BaseUrl}/tickets/{id}`. `QueueReceivedAsync` / `QueueResolvedAsync` / `QueueVerificationCodeAsync` use `CustomerTemplates` (customer `PreferredLanguage` en/ar, RTL HTML, organization name and `PrimaryColor`, HTML-encoded).
- `ChannelDeliveryHandlers` (165–229) on `DomainEventNotification<…>`: public agent message → `QueueAgentReplyAsync` (+ chat push, CH-02); ticket created on Email/WebForm/Portal/WhatsApp/Sms → "received"; status → `Resolved` on non-chat tickets → "resolved".
- `DispatchOutboxHandler` (236–283): 50 due `Pending` rows ordered by `NextAttemptAt`; missing/unconfigured sender → permanent failure; transport exceptions (`HttpRequestException`, `TimeoutException`, `SmtpException`, `IOException`) → retryable; `SaveChangesAsync` after each row so a crash never re-sends.

### 7 — Administration endpoints

`ChannelAdministrationEndpoints : IEndpoint` group `/channels`, `.RequireAuthorization(Permissions.ChannelsManage)`, tag `Channels`:

- `GET /status` → `ChannelStatusResponse(Channel, Configured)[]`.
- `POST /test` `TestChannelRequest(Channel, To)` → `SendChannelTestCommand` (validator: channel `Email|WhatsApp|Sms`, `To` ≤ 320) → queued row, `ApiResults.Success()`.
- `GET /outbox?status=&page=` → `OutboundMessageResponse` list, page size fixed at 50, `PaginationMeta`.
- `POST /outbox/{id:guid}/retry` → `Retry(now)`; unknown id → 404 `OUTBOUND_MESSAGE_NOT_FOUND`.

### 8 — Configuration and localization

- `src/CustomerSupportCrm.Api/appsettings.json` at `2956767` lines 60–86 — `Channels` sections with `Enabled: false` and empty credentials, `Portal:BaseUrl` (section added in `678ea67`). Real values come from environment / user-secrets. `Channels:Email:InboundSecret` and `Channels:Sms:InboundWebhookUrl` are not listed there.
- No resx keys added in `e921626`. Commit `0936711` added feature error keys, but `NO_ACTIVE_BRANCH`, `OUTBOUND_MESSAGE_NOT_FOUND` have no entry in `Messages.resx` / `Messages.ar.resx` (fallback English message is returned).

---

## Edge Cases & Failure Modes

- **Duplicate webhook delivery** — receipt check returns `null`; a concurrent duplicate hits the unique index → 409 `CONFLICT` to the provider, which retries and is then deduped.
- **Inbound secret / app secret / auth token not configured** — adapter returns `null` → 401 on every POST (fail closed).
- **Reply by email to a closed ticket or with a tag of another customer's ticket** — a new ticket is created.
- **Customer exists with phone, writes on WhatsApp** — matched (Phone/WhatsApp equivalence).
- **No active branch** — inbound processing / web form fails with 404 `NO_ACTIVE_BRANCH`.
- **Honeypot filled** — 400 `VALIDATION_ERROR`, field `form.website`, code `INVALID`.
- **Web form disabled** — 409 `FEATURE_DISABLED` (`678ea67`).
- **Rate limit** — web form / categories over 30 requests per minute per IP → 429 `RATE_LIMITED`. Webhooks have no dedicated limiter (global limiter from `2956767` only).
- **Provider down** — retry at 1, 2, 4, 8, 16 minutes; failed after 6 attempts; admin can retry.
- **Channel disabled in config** — queued rows fail permanently with "The X channel is not configured."
- **Agent reply on a Portal/Agent ticket without customer email** — nothing is queued.

---

## Test Plan

Test projects are **out of scope** (`tests/` must not be modified). No tests were added for this story, and no existing test in `tests/` covers channels, webhooks, web forms or the outbox (searched at `2956767`).

---

## Verification Steps

1. **Backend builds:** in `customer-support-crm-api/` run `dotnet build` — 0 warnings, 0 errors.
2. **Web form:** `curl -i -X POST https://localhost:<port>/api/v1/public/web-forms/tickets -H "Content-Type: application/json" -d '{"name":"Jane","email":"jane@example.com","subject":"Help","message":"Hi","website":""}'` → 201 `{ticketNumber}`; with `"website":"x"` → 400.
3. **Categories:** `curl …/api/v1/public/web-forms/categories` → active categories with `nameAr`.
4. **Email webhook:** set `Channels__Email__InboundSecret=s3cret`; `curl -X POST …/api/v1/public/channels/email/webhook -H "X-Webhook-Secret: s3cret" -H "Content-Type: application/json" -d '{"messageId":"m1","from":"Jane <jane@example.com>","subject":"Order","text":"Where is it?"}'` → 200 and a new `Email` ticket; repeat → no second ticket; reply with subject `Re: [T-000001] Order` → message added to that ticket; wrong secret → 401.
5. **Unknown channel:** `POST …/channels/fax/webhook` → 404.
6. **Outbox:** with an admin token `curl …/api/v1/channels/outbox -H "Authorization: Bearer $TOKEN"` → "received" notice rows; with channels disabled they become `Failed`; `POST …/channels/outbox/{id}/retry` → `Pending`.
7. **Status / test:** `GET …/api/v1/channels/status` → three rows; `POST …/channels/test -d '{"channel":"Fax","to":"x"}'` → 400 `INVALID_VALUE`.
8. **Permissions:** an Agent token on `GET /api/v1/channels/status` → 403.

---

## Done Criteria

- [x] Provider-neutral `IMessageSender` and `IChannelWebhookAdapter`; provider code only in Infrastructure. (`ICommunicationChannel` / `IMessageReceiver` not built — see deviations.)
- [x] Inbound messages normalized to `InboundChannelMessage`, matched to customer by email/phone and to ticket by subject tag or 7-day conversation window.
- [x] Unknown senders create a customer. (Not configurable; no lead option.)
- [x] Webhooks verified (shared secret / HMAC) and deduplicated by provider message id.
- [x] Outbound delivery through an outbox with a recurring dispatcher, exponential backoff and `Pending/Sent/Failed` tracking. (No provider delivery receipts.)
- [x] Web form rate-limited with a honeypot; categories endpoint for the form (`c8e9251`).
- [x] Credentials read from configuration; `appsettings.json` holds empty placeholders only.
- [ ] Channel configuration per branch/department (`ChannelConfiguration` CRUD) — not built; options only.
- [ ] IMAP polling and WhatsApp templates — not built.
- [ ] Localized messages for `NO_ACTIVE_BRANCH`, `OUTBOUND_MESSAGE_NOT_FOUND` — missing in resx.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/` by this story.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to Story 33.**
