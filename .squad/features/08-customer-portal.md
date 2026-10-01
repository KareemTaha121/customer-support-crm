# 08 — Customer Portal

> **Source:** AZM Squad Customer Support CRM — Core Features §8
> **Implementation phase:** Phase 9 — Customer Portal
> **Status:** Done (backend + frontend) — backend plans [30–31, 40](../plans/customer-portal/00-overview.md), frontend plan [17](../plans/frontend/17-story-customer-portal-ui.md)
> **Build priority:** 12

## Summary

A self-service area where customers submit and follow their requests, read help content, and tell us how we did.

## Scope

| Sub-feature | Description |
|-------------|-------------|
| Submit tickets | Create a request with category, description, attachments. |
| Track requests | See status and reply on open tickets. |
| View history | List of past tickets and conversations. |
| Access FAQs | Browse/search public knowledge base. |
| Submit feedback | CSAT rating and comment after resolution. |

## User stories

- As a **customer**, I register / sign in (email or phone OTP) to the portal.
- As a **customer**, I submit a ticket and receive a ticket number.
- As a **customer**, I see the status of my tickets and reply to the agent.
- As a **customer**, I view my past tickets.
- As a **customer**, I search FAQs before submitting a ticket.
- As a **customer**, I rate my experience when a ticket is resolved.

## Acceptance criteria

- [ ] Reuses the same backend domain and API contracts; separate customer authorization policies.
- [ ] Customer sees **only their own** tickets (or their company's, per role).
- [ ] Internal notes, internal articles, assignment details and agent-only data are never exposed.
- [ ] Only public ticket statuses shown (mapped customer-friendly labels).
- [ ] CSAT survey triggered on `TicketResolved`; one response per ticket.
- [ ] Deflection: suggested FAQs shown while typing the ticket subject.
- [ ] Branding per organization (logo, colors) and AR/EN with RTL.
- [ ] Mobile-friendly.

## Domain model

```text
CustomerUser (portal identity linked to Customer)
CustomerFeedback (Rating, Comment, TicketId)
```

## Backend slices

```text
Features/Portal/
├── Auth/ (Register, SignIn, VerifyOtp)
├── Tickets/ (Submit, ListMine, GetMine, Reply)
├── KnowledgeBase/ (Browse, Search)
└── Feedback/ (Submit)
```

## Frontend

- Separate portal area/layout (or app) with its own routes and guards.

## Dependencies

- Ticket Management (02), Knowledge Base (06), Security (10), Platform branding (12). Feeds: Reports CSAT (09).
