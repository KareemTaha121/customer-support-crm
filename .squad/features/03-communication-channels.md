# 03 — Communication Channels

> **Source:** AZM Squad Customer Support CRM — Core Features §3
> **Implementation phase:** Phase 10 — Communications
> **Status:** Done (backend + frontend) — backend plans [32–33](../plans/communication-channels/00-overview.md), frontend plan [12](../plans/frontend/12-story-channels-and-live-chat-console.md)
> **Build priority:** 13

## Summary

Let customers reach support through the channel they prefer, and let agents answer from one place. Every inbound message is normalized into the same ticket/message model, regardless of provider.

## Scope

| Channel | Inbound | Outbound |
|---------|---------|----------|
| Email | Mailbox polling / inbound webhook → ticket or reply | Replies from ticket, templates |
| WhatsApp | WhatsApp Business API webhook | Replies, approved templates |
| Live chat | Embeddable web widget (real-time) | Agent chat console |
| SMS | Provider webhook | Replies, notifications |
| Web forms | Public form → ticket | Confirmation to customer |

## User stories

- As a **customer**, I can email/WhatsApp/SMS/chat support and my message becomes a ticket automatically.
- As a **customer**, my replies on the same channel are threaded into my existing ticket.
- As an **agent**, I reply from the ticket and the reply goes out on the customer's channel.
- As an **admin**, I can configure channel accounts (mailboxes, numbers, widget) per branch/department.

## Acceptance criteria

- [ ] Provider-neutral abstractions: `ICommunicationChannel`, `IMessageSender`, `IMessageReceiver`.
- [ ] Provider SDKs only in Infrastructure; no per-provider ticket business logic.
- [ ] Inbound messages normalized to a common `InboundMessage` model and matched to customer (by email/phone) and ticket (by thread/reference).
- [ ] Unknown senders create a new customer (or lead) record per configuration.
- [ ] Webhooks validated (signatures), idempotent (deduplicate by provider message id).
- [ ] Outbound sending runs via background jobs with retries and delivery status tracking.
- [ ] Live chat uses real-time transport (e.g. SignalR); chat transcript stored as ticket messages.
- [ ] Web forms protected against spam/abuse (rate limiting).
- [ ] Channel credentials stored as secrets, never in source.

## Domain model

```text
ChannelConfiguration
TicketMessage (Channel, Direction, ProviderMessageId, DeliveryStatus)
```

## Backend slices

```text
Features/Channels/
├── Configurations/ (CRUD, Test connection)
├── Inbound/ (ReceiveEmail, ReceiveWhatsApp, ReceiveSms, ReceiveWebForm)
├── LiveChat/ (StartSession, SendMessage, EndSession)
└── Outbound/ (SendReply, TrackDelivery)
Infrastructure/Channels/ (Smtp/Imap, WhatsApp, Sms, SignalR adapters)
```

## Frontend

- Channel badges and per-channel composer in ticket details.
- Live chat agent console.
- Admin screens for channel configuration.
- Embeddable chat widget and public web form.

## Dependencies

- Ticket Management (02), Customer Management (01), Integrations (11), Platform background jobs.
