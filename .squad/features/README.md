# Features

Feature breakdown of the **AZM Squad Customer Support CRM**, one file per feature from the product requirements PDF (`azm_squad_customer_support_crm.pdf`). Phases and build priority follow `customer-support-crm-implementation-plan.md`.

Use each file as input for `squad new-story <feature-slug>`: split its user stories into story intakes under `.squad/stories/`.

| # | Feature | Sub-features | Phase | Priority |
|---|---------|--------------|-------|----------|
| 01 | [Customer Management](01-customer-management.md) | Profiles, contacts, interaction history, notes & attachments | 4 | 4 |
| 02 | [Ticket Management](02-ticket-management.md) | Create/track, categories & priorities, assignment, status & escalation, history | 5 | 5–7 |
| 03 | [Communication Channels](03-communication-channels.md) | Email, WhatsApp, live chat, SMS, web forms | 10 | 13 |
| 04 | [Agent Dashboard](04-agent-dashboard.md) | Assigned tickets, customer info, tasks & reminders, quick replies, collaboration | 6 | 8 |
| 05 | [SLA & Automation](05-sla-and-automation.md) | Targets, auto-assignment, escalation rules, alerts & notifications | 7 | 9–10 |
| 06 | [Knowledge Base](06-knowledge-base.md) | FAQs, help articles, solutions & guides, search | 8 | 11 |
| 07 | [AI Features](07-ai-features.md) | Summaries, suggested replies, categorization, suggested solutions, chatbot | 12 | 15 |
| 08 | [Customer Portal](08-customer-portal.md) | Submit, track, history, FAQs, feedback | 9 | 12 |
| 09 | [Reports & Management](09-reports-and-management.md) | Ticket, SLA, agent, CSAT reports, management dashboards | 11 | 14 |
| 10 | [Security & Administration](10-security-and-administration.md) | Users & roles, permissions, audit logs, system configuration | 2–3 | 2–3 |
| 11 | [Integrations](11-integrations.md) | APIs, ERP, email/SMS/WhatsApp providers, external systems | 13 | 16 |
| 12 | [Platform](12-platform.md) | Arabic & English, responsive, multi-department, multi-branch, branding | 0–1, 3 | 1 |

## Suggested build order

1. **12** Platform → 2. **10** Security & Administration → 3. **01** Customers → 4. **02** Tickets → 5. **04** Agent Dashboard → 6. **05** SLA & Automation → 7. **06** Knowledge Base → 8. **08** Customer Portal → 9. **03** Communication Channels → 10. **09** Reports → 11. **07** AI → 12. **11** Integrations

## Dependency map

```text
12 Platform
 └── 10 Security & Administration
      ├── 01 Customers
      │    └── 02 Tickets
      │         ├── 04 Agent Dashboard
      │         ├── 05 SLA & Automation
      │         ├── 03 Communication Channels ── 11 Integrations
      │         └── 09 Reports
      └── 06 Knowledge Base
           ├── 08 Customer Portal (+ 02)
           └── 07 AI Features (+ 02, 03)
```

Every feature must satisfy the **Feature Definition of Done** (implementation plan §80).

## Plans and stories

Every feature has a backend plan, a frontend plan and story intakes. Plans live in `.squad/plans/<feature>/`, intakes in `.squad/stories/<feature>/<story>/intake.md`. Global `NN` order is in [../plans/00-index.md](../plans/00-index.md).

| # | Feature | Backend plans | Frontend plan | Status |
|---|---------|---------------|---------------|--------|
| 01 | Customer Management | [23–24](../plans/customer-management/00-overview.md) | [10](../plans/frontend/10-story-customers-ui.md) | Done |
| 02 | Ticket Management | [25–26](../plans/ticket-management/00-overview.md) | [11](../plans/frontend/11-story-tickets-ui.md) | Done |
| 03 | Communication Channels | [32–33, 37](../plans/communication-channels/00-overview.md) | [12](../plans/frontend/12-story-channels-and-live-chat-console.md) | Done |
| 04 | Agent Dashboard | [27, 38](../plans/agent-dashboard/00-overview.md) | [13](../plans/frontend/13-story-agent-dashboard-ui.md) | Done |
| 05 | SLA & Automation | [28, 39](../plans/sla-and-automation/00-overview.md) | [14](../plans/frontend/14-story-sla-and-automation-admin-ui.md) | Done |
| 06 | Knowledge Base | [29](../plans/knowledge-base/00-overview.md) | [15](../plans/frontend/15-story-knowledge-base-ui.md) | Done |
| 07 | AI Features | [35](../plans/ai-features/00-overview.md) | [16](../plans/frontend/16-story-ai-assistant-panels.md) | Done |
| 08 | Customer Portal | [30–31, 40](../plans/customer-portal/00-overview.md) | [17](../plans/frontend/17-story-customer-portal-ui.md) | Done |
| 09 | Reports & Management | [34](../plans/reports-and-management/00-overview.md) | [18](../plans/frontend/18-story-reports-ui.md) | Done |
| 10 | Security & Administration | [01–07, 22](../plans/security-and-administration/00-overview.md) | [09](../plans/frontend/09-story-authentication-staff-and-portal.md), [19](../plans/frontend/19-story-administration-ui.md) | Done |
| 11 | Integrations | [36](../plans/integrations/00-overview.md) | [19](../plans/frontend/19-story-administration-ui.md) | Done |
| 12 | Platform | [20–21](../plans/platform/00-overview.md) | [08](../plans/frontend/08-story-core-platform-shell.md), [19](../plans/frontend/19-story-administration-ui.md) | Done |

Backend plans 20–36 are as-built (written after the code). Each one has a deviations table listing where the code differs from this feature spec.
