# Communication Channels — feature 03

Feature spec: [../../features/03-communication-channels.md](../../features/03-communication-channels.md).
Repo: `customer-support-crm-api`. Out of scope for every story: Docker, deployment, CI/CD, and files under `tests/`.

| NN | Tracker id | Plan file | Title | Depends on | Status |
|----|-----------|-----------|-------|-----------|--------|
| 32 | CH-01 | [32-story-inbound-channels-and-outbound-messaging.md](32-story-inbound-channels-and-outbound-messaging.md) | Inbound channels (email / WhatsApp / SMS / web form) and outbound customer messaging | 20–21, 23–26 | Done |
| 33 | CH-02 | [33-story-live-chat.md](33-story-live-chat.md) | Live chat (visitor API, agent handling, realtime, transcript) | 32, 20–21, 25–26 | Done |
| 37 | BUG-01 | [37-story-staff-hub-conversation-access.md](37-story-staff-hub-conversation-access.md) | Staff hub: check permission and scope before joining a chat conversation | 33 | Done (`d2563dc`) |
| 49 | BUG-13 | [49-story-outbound-delivery-visibility.md](49-story-outbound-delivery-visibility.md) | Show staff when a customer reply was not delivered; dev Log email provider (QA H3, round 2 N1) | 32 | Done (api `b94f982`, web `a28cb1b`) |
| 53 | BUG-17 | [53-story-live-chat-opening-message.md](53-story-live-chat-opening-message.md) | Show the visitor's opening message in both live chat transcripts (QA M4) | 33, 50 | To do |

Story intakes: [`../../stories/communication-channels/`](../../stories/communication-channels/).
Frontend counterpart: [../frontend/12-story-channels-and-live-chat-console.md](../frontend/12-story-channels-and-live-chat-console.md) (FE-05 — channel badges, live chat console, widget, public web form, channel admin).

Both stories were implemented in `customer-support-crm-api` commit `e921626` (feat: add communication channels (feature 03)) on `develop`, before these plans were written; both plans are **as-built**. Follow-ups touching the feature:

- `678ea67` — migration `20260930104602_AddSupportOperations` creates `outbound_messages`, `inbound_receipts`, `chat_conversations` (no migration shipped in `e921626`); `Channels` / `Portal` sections in `appsettings.json`; settings toggles `chat.enabled` and `webform.enabled` enforced by `FeatureToggleBehavior`.
- `0936711` — `CHAT_NOT_FOUND` localized (en/ar).
- `f6074d2` — `GET /api/v1/chat/conversations/{id}/messages` staff transcript (`chat.handle`).
- `c8e9251` — `GET /api/v1/public/web-forms/categories` for the contact form.

Plan 32 line numbers refer to `c8e9251`, plan 33 to `f6074d2`; both trees are identical to `2956767` for the cited channel files.

Known gaps and drift:

- **No channel configuration entity or CRUD.** The spec's `ChannelConfiguration` (accounts per branch/department) was not built; one email / WhatsApp / SMS account per deployment, configured through options (`Channels:*`). `GET /channels/status` only reports whether each sender is configured.
- **Abstractions differ from the spec.** `IMessageSender` + `IChannelWebhookAdapter` (the latter declared in the Application feature file, not `Abstractions/`); no `ICommunicationChannel` / `IMessageReceiver`; normalized model is `InboundChannelMessage`.
- **No IMAP polling, no WhatsApp templates, no provider delivery receipts.** Delivery status is outbox-level (`Pending`/`Sent`/`Failed`, `ProviderMessageId`), not on `TicketMessage`. WhatsApp free text only works inside the 24-hour customer-service window.
- **Email inbound uses a shared secret header**, not a provider signature. Webhooks have no dedicated rate limiter.
- **Unknown senders always become customers** (`auto-created` tag); there is no lead option or setting.
- ~~**`StaffHub.JoinConversation` had no permission/scope check** (any staff connection could follow any conversation id, hub from `65c74a3`)~~ — fixed by story 37 (`d2563dc`).
- **`StartChatRequest.Language` is ignored.**
- **Missing resx keys:** `CHAT_CLOSED`, `NO_ACTIVE_BRANCH`, `OUTBOUND_MESSAGE_NOT_FOUND` → BUG-09 ([intake](../../stories/platform/missing-error-messages/intake.md)).
- **No automated tests** for channels or live chat in `tests/`.
- `Channels:Email:InboundSecret` and `Channels:Sms:InboundWebhookUrl` are required for inbound email / SMS but are not listed in `appsettings.json`.
