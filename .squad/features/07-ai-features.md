# 07 — AI Features

> **Source:** AZM Squad Customer Support CRM — Core Features §7
> **Implementation phase:** Phase 12 — AI
> **Status:** Done (backend + frontend) — backend plans [35](../plans/ai-features/00-overview.md), frontend plan [16](../plans/frontend/16-story-ai-assistant-panels.md)
> **Build priority:** 15

## Summary

Use AI to make agents faster and help customers self-serve: summarize, suggest, categorize and chat — always with a human in control.

## Scope

| Sub-feature | Description |
|-------------|-------------|
| Ticket summaries | Summarize a long ticket conversation for the agent/supervisor. |
| Suggested replies | Draft a reply (AR/EN) the agent can edit before sending. |
| Automatic categorization | Suggest category/priority/department for new tickets. |
| Suggested solutions | Recommend relevant KB articles and similar resolved tickets. |
| AI chatbot | Answer customers on portal/live chat from the KB; hand off to a human. |

## User stories

- As an **agent**, I click "Summarize" and get a short summary of the ticket.
- As an **agent**, I get a suggested reply that I can edit, accept or discard.
- As a **supervisor**, new tickets arrive pre-categorized with a confidence score.
- As an **agent**, I see suggested KB articles for the current ticket.
- As a **customer**, I chat with a bot that answers from the KB and transfers me to an agent when needed.

## Acceptance criteria

- [ ] AI is isolated behind `IAiService` at the Application boundary; provider SDKs only in Infrastructure; nothing in Domain.
- [ ] AI output treated as untrusted content; never auto-executes high-impact actions (send, close, reassign) without human confirmation or explicit rules.
- [ ] Suggestions stored as `AiSuggestion` with accepted/rejected feedback for quality tracking.
- [ ] Customer PII handling and data-retention rules documented; configurable per organization.
- [ ] Chatbot grounded on published public KB only; clear hand-off to live agent creates/updates a ticket.
- [ ] Feature flags to enable/disable each AI capability per organization.
- [ ] Timeouts, rate limits and cost tracking for provider calls.
- [ ] Arabic and English support.

## Domain model

```text
AiSuggestion (Type, Input ref, Output, Confidence, Status: Pending/Accepted/Rejected)
AiConversation (chatbot sessions)
```

## Backend slices

```text
Application/Ai/
├── SummarizeTicket
├── SuggestReply
├── CategorizeTicket
├── SuggestSolution
└── Chatbot
Infrastructure/Ai/ (provider implementation of IAiService)
```

## Frontend

- AI actions in ticket details (summary card, suggested reply in composer, suggested articles).
- Chatbot mode in live chat widget / portal.
- Admin: AI feature toggles.

## Dependencies

- Ticket Management (02), Knowledge Base (06), Communication Channels (03) live chat, Customer Portal (08).
