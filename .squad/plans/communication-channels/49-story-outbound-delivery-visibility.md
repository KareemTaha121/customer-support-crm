# Story 49 — Show staff when a reply to the customer was not delivered (Bug: BUG-13)

> Implemented in `customer-support-crm-api` commit `b94f982` (`develop`) and `customer-support-crm-web` commit `a28cb1b` (`main`) on 2026-10-01. The plan was written against api `dca992c` / web `06c817a`; its line numbers refer to those commits. See *Implementation notes* at the end for additions.
> Intake: [../../stories/communication-channels/outbound-delivery-visibility/intake.md](../../stories/communication-channels/outbound-delivery-visibility/intake.md)

## Prerequisites

- Story 32 — [32-story-inbound-channels-and-outbound-messaging.md](32-story-inbound-channels-and-outbound-messaging.md): `OutboundMessage`, `CustomerMessenger`, `DispatchOutboxHandler`, Channels admin endpoints.
- Frontend story 11 — [../frontend/11-story-tickets-ui.md](../frontend/11-story-tickets-ui.md): `TicketConversationComponent`.
- Frontend story 12 — [../frontend/12-story-channels-and-live-chat-console.md](../frontend/12-story-channels-and-live-chat-console.md): `ChannelsAdminPage`.
- Source: manual QA report `.squad/qa/2026-10-01-manual-qa-report.md`, finding H3.
- Backend paths are relative to **`customer-support-crm-api/`**, frontend paths to **`customer-support-crm-web/`**.

---

## Story Goal

An agent's public reply raises `TicketMessageAddedDomainEvent`. `ChannelDeliveryHandlers` then calls `CustomerMessenger.QueueAgentReplyAsync`, which adds an `OutboundMessage` with `TicketId` **and** `TicketMessageId` set, in the same `SaveChanges`. Every 15 seconds `DispatchOutboxHandler` sends due rows. If the channel's sender is missing or not configured, it marks the row `Failed` at once (`permanent: true`, attempts 1). The link between reply and delivery already exists; nothing reads it outside the Channels page.

| Surface | Before | After |
|---|---|---|
| `GET /tickets/{id}/messages`, `POST /tickets/{id}/messages` | No delivery information | `delivery` block on agent replies that were queued |
| Staff conversation | Reply looks delivered | Chip: Queued / Sent / Not delivered, with Retry for `channels.manage` |
| Reply toast | Always "Reply sent." | Warning when the delivery channel is not configured |
| Channels page | Failed rows only visible in the table | Banner "N messages could not be delivered" + "Show failed" |
| `POST /channels/outbox/{id}/retry` on a `Sent` row | Re-queues and re-sends | 409 `OUTBOUND_ALREADY_SENT` |
| Development without SMTP | Every email fails "not configured" | Optional `Log` provider: logged and marked `Sent` |

**Scope choice for the banner (intake item 3):** the Channels page, not the staff shell or dashboard. The shell is in `core/` and shared by every user, while failures are actionable only by `channels.manage` holders, who already use that page. Agents see failures on the ticket itself (chip and toast).

**Deviation from the intake:** none.

---

## Context — Read These Files First

Backend (`customer-support-crm-api/`):

1. `src/CustomerSupportCrm.Domain/Tickets/ChannelRecords.cs`:
   - `OutboundStatus` `Pending | Sent | Failed` (7–12);
   - `OutboundMessage` (18) with `MaxAttempts = 6` (20), `TicketId` (45), `TicketMessageId` (47);
   - `Queue` (63);
   - `MarkSent` (88);
   - `MarkFailed` (98), with backoff 1, 2, 4, 8, 16 minutes, and `Failed` when `permanent` or `Attempts >= 6`;
   - `Retry` (111): sets `Pending`, keeps `Attempts`, and has **no status check**.
2. `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/ChannelConfiguration.cs` lines 8–24: `outbound_messages`, index on `TicketId` (22). No index on `TicketMessageId`, but it is not needed: lookups filter by `TicketId` first.
3. `src/CustomerSupportCrm.Application/Features/Channels/CustomerMessaging.cs`:
   - `QueueAgentReplyAsync` (70–96): WhatsApp/SMS reply to the phone (80); `Chat` queues nothing (82–84); every other channel (Portal, WebForm, Email, Phone, Api, Agent) queues an **Email** with the reply, if the ticket or customer has an address (85–94);
   - `QueueNotice` (129–146): "received" and "resolved" notices, with `ticketMessageId: null`;
   - `ChannelDeliveryHandlers` (165–202): agent replies only (185–188);
   - `DispatchOutboxHandler` (236–283): "not configured" is a permanent failure (251–256); transient exceptions are retried (263–266);
   - `OutboundMessageResponse` (289);
   - Channels endpoints (314–361), all `channels.manage` (318); retry at 351–360.
4. `src/CustomerSupportCrm.Application/Abstractions/Channels/ChannelAbstractions.cs` lines 15–22: `IMessageSender { Channel; IsConfigured; SendAsync }`.
5. `src/CustomerSupportCrm.Contracts/Tickets/TicketContracts.cs` lines 101–111: `TicketMessageResponse`.
6. `src/CustomerSupportCrm.Application/Features/Tickets/Common/TicketQueries.cs` lines 176–216: `GetMessagesAsync(db, ticketId, publicOnly, downloadUrl, ct)`. Its callers:
   - staff: `TicketQuerySlices.cs` 194 (`GET /tickets/{id}/messages`, endpoint 243–246) and `TicketMessageSlices.cs` 106 (`POST` reply response, handler 77–108);
   - public: `PortalTicketSlices.cs` 121, and `LiveChat.cs` 179 and 325.
7. `src/CustomerSupportCrm.Infrastructure/Channels/MessageSenders.cs` lines 17–60: `SmtpEmailSender`, `IsConfigured` (23) = `Enabled && Host && FromAddress`.
8. `src/CustomerSupportCrm.Infrastructure/Channels/ChannelOptions.cs` lines 4–26: `EmailOptions` (`Channels:Email`).
9. `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` lines 107–122: sender registrations (114–116) and the outbox job every 15 s (122).
10. `src/CustomerSupportCrm.Api/appsettings.json` lines 60–67 (`Channels:Email`, `Enabled: false`, no `FromAddress`); `appsettings.Development.json` has no `Channels` section.
11. `src/CustomerSupportCrm.Application/Resources/Messages.resx` line 204 / `Messages.ar.resx` line 243: `OUTBOUND_MESSAGE_NOT_FOUND`, the model for the new code.

Frontend (`customer-support-crm-web/`):

12. `src/app/features/tickets/tickets.models.ts` lines 92–103: `TicketMessage`.
13. `src/app/features/tickets/tickets.api.ts`: `messages` (36–38), `addMessage` (77).
14. `src/app/features/tickets/ticket-conversation.component.ts`:
    - the message header (67–77), with the channel pill at 73–75;
    - `send()` (214–235), whose success toast `tickets.reply.sent` is at 226;
    - the `changed` output (174) makes the parent `refresh()` (`ticket-details.page.ts` 203–225; wired at `ticket-details.page.html` 79).
15. `src/app/features/channels/channels-admin.page.ts`: `loadOutbox` (110–124), `setStatusFilter` (126–130), `canRetry` (137–139), `retry` (152–168). `channels-admin.page.html`: the outbox section starts at 66. `channels.api.ts`: `outbox` (49–51), `retry` (53–55).
16. `src/app/core/permissions/permissions.ts` line 28: `channelsManage`.
17. i18n: `public/i18n/tickets/{en,ar}.json` → `tickets.reply.*`, `tickets.channel.*`; `public/i18n/channels/{en,ar}.json` → `channels.admin.*`.

---

## Backend Tasks

### 1 — Contract

`Contracts/Tickets/TicketContracts.cs`:

```csharp
/// <summary>Delivery of an agent reply through the outbox. LastError only for channels.manage.</summary>
public sealed record TicketMessageDeliveryResponse(
    Guid OutboundMessageId,
    string Channel,
    string Status,
    int Attempts,
    DateTimeOffset? SentAt,
    bool ChannelConfigured,
    string? LastError);
```

Add a last positional parameter to `TicketMessageResponse`: `TicketMessageDeliveryResponse? Delivery = null`. The default keeps every existing constructor call compiling.

### 2 — Read deliveries in `GetMessagesAsync`

In `TicketQueries.cs`, add:

```csharp
/// <summary>Staff view of reply deliveries; null for customer-facing lists.</summary>
public sealed record DeliveryView(Func<TicketChannel, bool> IsChannelConfigured, bool IncludeErrors);
```

Add an optional parameter `DeliveryView? deliveryView = null` to `GetMessagesAsync`. When it is set, after loading `messages`:

```csharp
var deliveries = deliveryView is null
    ? []
    : await db.OutboundMessages.AsNoTracking()
        .Where(o => o.TicketId == ticketId && o.TicketMessageId != null && messageIds.Contains(o.TicketMessageId))
        .OrderByDescending(o => o.CreatedAt)
        .Select(o => new { o.Id, o.TicketMessageId, o.Channel, o.Status, o.Attempts, o.SentAt, o.LastError })
        .ToListAsync(cancellationToken);
```

Map the newest row per message (`deliveries.FirstOrDefault(d => d.TicketMessageId == m.Id)`) to `TicketMessageDeliveryResponse`:

- `ChannelConfigured = deliveryView.IsChannelConfigured(d.Channel)`;
- `LastError = deliveryView.IncludeErrors ? d.LastError : null`.

Provider errors can carry hostnames or API responses, so only `channels.manage` holders see them. The others get the status and `ChannelConfigured`, which the UI turns into a localized reason.

Portal (`PortalTicketSlices.cs` 121) and live chat (`LiveChat.cs` 179, 325) keep calling without `deliveryView`, so `delivery` is always `null` there.

### 3 — Staff callers pass the view

Inject `IEnumerable<IMessageSender> senders` and `ICurrentUser currentUser` (already injected in `AddTicketMessageHandler`) into `GetTicketMessagesHandler` (`TicketQuerySlices.cs` 189) and `AddTicketMessageHandler` (`TicketMessageSlices.cs` 77). Both pass:

```csharp
new TicketQueries.DeliveryView(
    channel => senders.Any(s => s.Channel == channel && s.IsConfigured),
    currentUser.HasPermission(Permissions.ChannelsManage))
```

To avoid repeating this, add a static factory `DeliveryView.For(IEnumerable<IMessageSender>, ICurrentUser)` next to the record. Domain events are dispatched inside `SaveChangesAsync` before the base save (`Infrastructure/Persistence/ApplicationDbContext.cs` 47 and 54), so the reply's outbox row is already saved when `AddTicketMessageHandler` reloads messages (106). The POST response therefore carries `delivery.status = "Pending"` and the correct `channelConfigured`.

### 4 — Retry guard

- In the retry endpoint (`CustomerMessaging.cs` 353–355), after loading the row, reject sent messages with a 409. The check sits next to the existing `OUTBOUND_MESSAGE_NOT_FOUND` lookup; `OutboundMessage.Retry` stays unchanged.

```csharp
if (message.Status == OutboundStatus.Sent)
{
    throw new ConflictException("OUTBOUND_ALREADY_SENT", "This message was already sent.");
}
```
- Add `OUTBOUND_ALREADY_SENT` to `Messages.resx` ("This message was already sent.") and `Messages.ar.resx` ("تم إرسال هذه الرسالة بالفعل.").
- `Attempts` is not reset on retry (unchanged behaviour). A message that already used its 6 attempts gets one more try per manual retry. That is acceptable and is documented here only.

### 5 — Development "Log" email provider

- `EmailOptions`: add `public string Provider { get; init; } = "Smtp";` and `public const string LogProvider = "Log";`.
- New `Infrastructure/Channels/LogEmailSender.cs`, `internal sealed partial class LogEmailSender(IHostEnvironment environment, ILogger<LogEmailSender> logger) : IMessageSender`:
  - `Channel => TicketChannel.Email`;
  - `IsConfigured => environment.IsDevelopment()`, so in any other environment the channel reports "not configured" and messages fail exactly as today;
  - `SendAsync` logs `To`, `Subject` and the text `Body` with a `[LoggerMessage(Level = LogLevel.Information, Message = "Email (log provider) to {To}: {Subject}\n{Body}")]` method, then returns `SendResult.Sent($"log-{Guid.CreateVersion7()}")`.
- `DependencyInjection.cs` line 114: replace the SMTP registration with a single factory, so `GET /channels/status` and the dispatcher still see exactly one Email sender:

```csharp
services.AddSingleton<IMessageSender>(sp =>
    string.Equals(sp.GetRequiredService<IOptions<EmailOptions>>().Value.Provider, EmailOptions.LogProvider, StringComparison.OrdinalIgnoreCase)
        ? ActivatorUtilities.CreateInstance<LogEmailSender>(sp)
        : ActivatorUtilities.CreateInstance<SmtpEmailSender>(sp));
```

- `appsettings.json` `Channels:Email`: add `"Provider": "Smtp"`. `appsettings.Development.json`: add `"Channels": { "Email": { "Provider": "Log" } }`. Development then delivers by log, and production is unchanged.

**No changes to:** domain entities, migrations (`TicketMessageId` already exists), the dispatcher, `CustomerMessenger`, docs.

---

## Frontend Tasks

### 6 — Model and API

`tickets.models.ts`:

```ts
/** TicketContracts.cs TicketMessageDeliveryResponse */
export interface TicketMessageDelivery {
  outboundMessageId: string;
  channel: string;
  status: 'Pending' | 'Sent' | 'Failed';
  attempts: number;
  sentAt: string | null;
  channelConfigured: boolean;
  lastError: string | null;
}
```

Add `delivery: TicketMessageDelivery | null;` to `TicketMessage`.

`tickets.api.ts`: add `retryDelivery(outboundMessageId: string): Observable<null>` → `this.api.post<null>(`/channels/outbox/${outboundMessageId}/retry`)`. The call stays in the tickets feature, so the conversation does not import the channels feature.

### 7 — Delivery chip in the conversation

In `ticket-conversation.component.ts`, in the message header after the channel pill (73–75), for `!message.isInternal && message.delivery`:

```html
@if (!message.isInternal && message.delivery; as d) {
  <span [class]="deliveryClass(d)" [matTooltip]="deliveryTooltip(d)">
    <mat-icon>{{ d.status === 'Sent' ? 'done_all' : !d.channelConfigured || d.status === 'Failed' ? 'error_outline' : 'schedule' }}</mat-icon>
    {{ deliveryLabel(d) | t }}
  </span>
  @if (canRetryDelivery && d.status === 'Failed') {
    <button mat-button type="button" (click)="retryDelivery(d)" [disabled]="retrying() === d.outboundMessageId">
      <mat-icon>replay</mat-icon>{{ 'tickets.delivery.retry' | t }}
    </button>
  }
}
```

- `deliveryLabel`: `Sent` → `tickets.delivery.Sent`; `Failed`, or `Pending` with `!channelConfigured` → `tickets.delivery.Failed`; otherwise `tickets.delivery.Pending`.
- `deliveryClass`: `crm-pill crm-pill--success` / `--danger` / `--info`, following `ChannelsAdminPage.statusClass` (141–150).
- `deliveryTooltip`:
  - `!channelConfigured` → `tickets.delivery.notConfigured` with `{ channel: t('tickets.channel.' + d.channel) }`;
  - else `Failed` → `d.lastError` or `tickets.delivery.failedHint`;
  - `Sent` → the localized `sentAt`;
  - else empty.
- `readonly canRetryDelivery = inject(PermissionService).has(Permissions.channelsManage);` and `readonly retrying = signal<string | null>(null);`.
- `retryDelivery(d)`: call `api.retryDelivery`, then `toast.success('tickets.delivery.retried')` and `this.changed.emit()` (the parent reloads); on error clear `retrying` (the global snackbar shows the error).

The delivery channel can differ from the message channel: a Portal ticket reply is delivered as an Email notification. The tooltip names the delivery channel.

### 8 — Reply toast

In `send()` (`next`, line 222), use the returned message:

```ts
next: (message) => {
  ...
  if (!message.isInternal && message.delivery && !message.delivery.channelConfigured) {
    this.toast.error('tickets.reply.sentNotDelivered', { channel: this.translations.t('tickets.channel.' + message.delivery.channel) });
  } else {
    this.toast.success(this.internal() ? 'tickets.reply.noteAdded' : 'tickets.reply.sent');
  }
  this.changed.emit();
},
```

`NotificationToastService` has `success`, `info` and `error` (11–19), but no `warning`. `error` is used because the customer did not get the message.

### 9 — Channels page banner

In `channels-admin.page.ts`, add `readonly failedCount = signal(0);` and `loadFailedCount()`: `this.api.outbox('Failed', 1)` → `failedCount.set(result.meta.totalCount)`; on error, `0`. Call it from the constructor, after `retry` succeeds, and from the outbox refresh. In `channels-admin.page.html`, above the outbox section (66), when `failedCount() > 0`:

```html
<div class="failed-banner crm-card" role="status">
  <mat-icon>error_outline</mat-icon>
  <span>{{ 'channels.admin.failedBanner' | t: { count: failedCount() } }}</span>
  <button mat-button type="button" (click)="setStatusFilter('Failed')">{{ 'channels.admin.showFailed' | t }}</button>
</div>
```

Style `.failed-banner` in `channels-admin.page.scss` with `display: flex; gap: 8px; align-items: center; color: var(--crm-danger);`.

### 10 — i18n (en / ar)

`public/i18n/tickets/{en,ar}.json`:

| Key | en | ar |
|---|---|---|
| `tickets.delivery.Pending` | Queued | في قائمة الإرسال |
| `tickets.delivery.Sent` | Sent | أُرسلت |
| `tickets.delivery.Failed` | Not delivered | لم تُرسل |
| `tickets.delivery.notConfigured` | The {{channel}} channel is not configured, so the customer has not received this reply. | قناة {{channel}} غير مُهيأة، لذلك لم يستلم العميل هذا الرد. |
| `tickets.delivery.failedHint` | Delivery failed. Ask an administrator to check the Channels page. | فشل الإرسال. اطلب من المسؤول مراجعة صفحة القنوات. |
| `tickets.delivery.retry` | Retry delivery | إعادة محاولة الإرسال |
| `tickets.delivery.retried` | The reply will be sent again shortly. | سيُعاد إرسال الرد قريبًا. |
| `tickets.reply.sentNotDelivered` | Reply saved, but the {{channel}} channel is not configured: the customer will not receive it. | تم حفظ الرد، لكن قناة {{channel}} غير مُهيأة ولن يستلمه العميل. |

`public/i18n/channels/{en,ar}.json`:

| Key | en | ar |
|---|---|---|
| `channels.admin.failedBanner` | {{count}} outgoing messages could not be delivered. | تعذّر إرسال {{count}} من الرسائل الصادرة. |
| `channels.admin.showFailed` | Show failed | عرض الرسائل الفاشلة |

---

## Test Plan

Test projects are **out of scope**: `tests/` must not be modified, and no e2e tests are added.

---

## Verification Steps

1. **Backend build:** `dotnet build src/CustomerSupportCrm.Api/CustomerSupportCrm.Api.csproj` gives 0 warnings and 0 errors. If a running API locks `bin/`, build with `-o <temp dir>`.
2. **Frontend build:** `cd customer-support-crm-web && npx ng build` gives 0 errors and 0 warnings.
3. **Not configured** (Development with `Channels:Email:Provider` set back to `Smtp` and no host):
   - post a public reply on a Portal or Email ticket → error toast "Reply saved, but the Email channel is not configured…";
   - the message shows "Not delivered", and the tooltip names the channel;
   - `GET /api/v1/tickets/{id}/messages` → `delivery.status` is `Pending`, then `Failed` after ≤ 15 s, with `channelConfigured: false`.
4. **Error visibility:** as an agent without `channels.manage`, `delivery.lastError` is `null` and there is no Retry button. As an admin, `lastError` is "The Email channel is not configured." and Retry is shown.
5. **Public lists:** `GET /api/v1/portal/tickets/{id}` messages and the live chat messages have `delivery: null`.
6. **Internal note / chat:** an internal note has `delivery: null`; a reply on a `Chat` ticket has `delivery: null` (nothing is queued).
7. **Log provider:** run in Development with `Provider: Log`. A reply → "Reply sent."; after ≤ 15 s the chip shows "Sent"; the API log shows the email; `GET /channels/status` → Email `configured: true`.
8. **Retry:** retry a failed delivery from the ticket → toast, and the row becomes `Pending`. `POST /api/v1/channels/outbox/{sentId}/retry` → 409 `OUTBOUND_ALREADY_SENT` (en/ar message).
9. **Channels page:** with failed rows, the banner shows the count; "Show failed" filters the table to `Failed`.
10. **Arabic:** switch to ar; check the chips, tooltips, toast and banner text, and that the layout is RTL.

---

## Done Criteria

- [x] Staff message lists and the reply response carry `delivery` on queued agent replies; portal and live chat lists never do.
- [x] `lastError` is visible only to `channels.manage` holders.
- [x] The conversation shows Queued / Sent / Not delivered chips in en and ar, with Retry for `channels.manage`.
- [x] A reply whose delivery channel is not configured shows a warning toast instead of "Reply sent.".
- [x] The Channels page shows the failed-message banner with "Show failed".
- [x] Retrying a sent message returns 409 `OUTBOUND_ALREADY_SENT`.
- [x] The `Log` email provider works only in Development and is off by default.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` passes with zero warnings; `ng build` passes with zero errors and zero warnings.

## Implementation notes (2026-10-01)

Built as planned. Additions:

| Addition | Why |
|---|---|
| The conversation reloads once, 20 s after it shows a `Queued` chip on a configured channel (`effect` in `TicketConversationComponent`) | The dispatcher runs every 15 s. Without the reload the chip stays "Queued" until the agent refreshes the page |
| `/admin/settings` warns on "Customer self-registration" when it is on and outgoing email is not configured (`AdministrationApi.emailConfigured()` → `GET /channels/status`, only for `channels.manage`; i18n `admin.settings.registrationNeedsEmail`) | QA round 2 finding **N1**: sign-up needs the emailed verification code |
| The Log provider logs the full body, so portal verification codes appear in the API log in Development | N1 in dev: register → read the code from the log → verify → sign in now works |
| The banner hides "Show failed" when the table is already filtered to Failed | Avoids a button that does nothing |
| `channels.admin.failedBanner` uses plural forms (en `one`/`other`, ar `zero`…`other`) | Arabic count agreement |

Verified at runtime (Chrome + API):
- Log provider: a reply goes `Pending` → `Sent`, and the email appears in the log. `GET /channels/status` reports Email `configured: true`.
- Retry: Retry on a failed reply shows a toast, then "Queued", then "Sent" after the automatic reload. Retrying a sent row returns 409 (en/ar).
- Error visibility: an agent without `channels.manage` gets `lastError: null`. Portal message lists have `delivery: null`; internal notes have `delivery: null`.
- Channels page: banner shows "4 outgoing messages could not be delivered.", and "Show failed" filters the table. In Arabic it reads "تعذّر إرسال 4 رسائل صادرة.".
- Not configured: a temporary API instance ran with `Channels__Email__Provider=Smtp`.
  - Reply: error toast "Reply saved, but the Email channel is not configured…".
  - Chip: "Not delivered", with a tooltip naming the channel.
  - The settings warning was shown.
- N1: `POST /public/portal/register` → the code appears in the log → `/verify` → `/login` succeeds.

**STOP HERE. Report to the user.**
