# Plans index

One row per feature folder under `.squad/plans/`. `NN` continues as a global execution sequence across all features when `naming.globalSequence` is `true` in `config.yaml`.

| Feature | Spec | Overview | NN range | Status |
|---------|------|----------|----------|--------|
| [security-and-administration](security-and-administration/00-overview.md) | [10](../features/10-security-and-administration.md) | Phase 2 identity & authorization (01–07) + system settings & audit export (22) | 01–07, 22 | Done |
| [frontend](frontend/00-overview.md) | all | Angular web app for all 12 features (staff app, customer portal, help center) | 08–19 | Done |
| [platform](platform/00-overview.md) | [12](../features/12-platform.md) | Backend foundation, organization context, branches/departments, notifications | 20–21 | Done |
| [customer-management](customer-management/00-overview.md) | [01](../features/01-customer-management.md) | Customer profiles, contacts, notes, attachments, interaction history | 23–24 | Done |
| [ticket-management](ticket-management/00-overview.md) | [02](../features/02-ticket-management.md) | Ticket lifecycle, assignment, escalation, conversation, categories | 25–26 | Done |
| [agent-dashboard](agent-dashboard/00-overview.md) | [04](../features/04-agent-dashboard.md) | Agent workspace: my tickets, tasks, quick replies, collaboration | 27 | Done |
| [sla-and-automation](sla-and-automation/00-overview.md) | [05](../features/05-sla-and-automation.md) | SLA policies, auto-assignment, escalation rules, breach alerts | 28 | Done |
| [knowledge-base](knowledge-base/00-overview.md) | [06](../features/06-knowledge-base.md) | Articles, categories, help center, search, ticket suggestions | 29 | Done |
| [customer-portal](customer-portal/00-overview.md) | [08](../features/08-customer-portal.md) | Portal accounts and auth, portal tickets, CSAT feedback | 30–31 | Done |
| [communication-channels](communication-channels/00-overview.md) | [03](../features/03-communication-channels.md) | Email / WhatsApp / SMS / web forms, outbound messaging, live chat | 32–33 | Done |
| [reports-and-management](reports-and-management/00-overview.md) | [09](../features/09-reports-and-management.md) | Ticket, SLA, agent, CSAT reports and management dashboards | 34 | Done |
| [ai-features](ai-features/00-overview.md) | [07](../features/07-ai-features.md) | Summaries, suggested replies, categorization, solutions, chatbot | 35 | Done |
| [integrations](integrations/00-overview.md) | [11](../features/11-integrations.md) | API keys, webhooks, providers, external systems | 36 | Done |

Backend plans 20–36 are **as-built**: the code was implemented on `customer-support-crm-api` `develop` (commits `c3c7815`, `65c74a3`…`678ea67` and follow-ups) before the plans were written. Each plan cites the commit its line numbers refer to and lists where the code differs from the feature spec.

Feature → plan coverage (backend + frontend) is in [../features/README.md](../features/README.md#plans-and-stories).
