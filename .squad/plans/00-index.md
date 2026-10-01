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

## Bug fixes (37–46)

Found while writing plans 20–36 (BUG-10 while fixing BUG-04). One intake per bug under the feature's `stories/` folder; the plan is written when the fix starts.

| NN | Id | Feature | Intake | Plan | Status |
|----|----|---------|--------|------|--------|
| 37 | BUG-01 | communication-channels | [staff-hub-conversation-access](../stories/communication-channels/staff-hub-conversation-access/intake.md) | [37](communication-channels/37-story-staff-hub-conversation-access.md) | Done (`d2563dc`) |
| 38 | BUG-02 | agent-dashboard | [task-scope-and-reference-validation](../stories/agent-dashboard/task-scope-and-reference-validation/intake.md) | [38](agent-dashboard/38-story-task-scope-and-reference-validation.md) | Done (`7058b62`) |
| 39 | BUG-03 | sla-and-automation | [ticket-sla-policy-foreign-key](../stories/sla-and-automation/ticket-sla-policy-foreign-key/intake.md) | [39](sla-and-automation/39-story-ticket-sla-policy-foreign-key.md) | Done (`3433f13`) |
| 40 | BUG-04 | customer-portal | [portal-profile-update-response](../stories/customer-portal/portal-profile-update-response/intake.md) | [40](customer-portal/40-story-portal-profile-update-response.md) | Done (`1ad5302`) |
| 41 | BUG-05 | customer-portal | [revoke-portal-sessions-on-access-revoke](../stories/customer-portal/revoke-portal-sessions-on-access-revoke/intake.md) | [41](customer-portal/41-story-revoke-portal-sessions-on-access-revoke.md) | Done (`31d6d5a`, `6587e7d`) |
| 42 | BUG-06 | platform | [inactive-organization-units](../stories/platform/inactive-organization-units/intake.md) | [42](platform/42-story-inactive-organization-units.md) | Done (`487e078`) |
| 43 | BUG-07 | ticket-management | [ticket-category-cycle-guard](../stories/ticket-management/ticket-category-cycle-guard/intake.md) | [43](ticket-management/43-story-ticket-category-cycle-guard.md) | Done (`cedce62`) |
| 44 | BUG-08 | knowledge-base | [knowledge-category-cycle-guard](../stories/knowledge-base/knowledge-category-cycle-guard/intake.md) | [44](knowledge-base/44-story-knowledge-category-cycle-guard.md) | Done (api `e6bf4d7`, web `06c817a`) |
| 45 | BUG-09 | platform | [missing-error-messages](../stories/platform/missing-error-messages/intake.md) | [45](platform/45-story-missing-error-messages.md) | Done (`5eb50e6`) |
| 46 | BUG-10 | customer-management | [staff-rename-portal-account-sync](../stories/customer-management/staff-rename-portal-account-sync/intake.md) | [46](customer-management/46-story-staff-rename-portal-account-sync.md) | Done (`dca992c`) |

Backend plans 20–36 are **as-built**: the code was implemented on `customer-support-crm-api` `develop` (commits `c3c7815`, `65c74a3`…`678ea67` and follow-ups) before the plans were written. Each plan cites the commit its line numbers refer to and lists where the code differs from the feature spec.

Feature → plan coverage (backend + frontend) is in [../features/README.md](../features/README.md#plans-and-stories).
