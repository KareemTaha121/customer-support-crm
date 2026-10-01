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

## QA fixes (47–57)

Found in the manual QA pass of 2026-10-01 ([report](../qa/2026-10-01-manual-qa-report.md)). Order, dependencies and the finding → story map: [qa-2026-10-01-fix-roadmap.md](qa-2026-10-01-fix-roadmap.md). Finding H4 (category cycles) was already fixed by plans 43–44.

| NN | Id | Feature | Intake | Plan | Status |
|----|----|---------|--------|------|--------|
| 47 | BUG-11 | security-and-administration | [password-reset](../stories/security-and-administration/password-reset/intake.md) | [47](security-and-administration/47-story-password-reset.md) | Done (api `ba569b9`, web `5ab9998`) |
| 48 | BUG-12 | ai-features | [hide-ai-when-unconfigured](../stories/ai-features/hide-ai-when-unconfigured/intake.md) | [48](ai-features/48-story-hide-ai-when-unconfigured.md) | Done (api `6e89942`, web `05d6969`) |
| 49 | BUG-13 | communication-channels | [outbound-delivery-visibility](../stories/communication-channels/outbound-delivery-visibility/intake.md) | [49](communication-channels/49-story-outbound-delivery-visibility.md) | Done (api `b94f982`, web `a28cb1b`; includes N1) |
| 50 | BUG-14 | frontend | [composer-error-state-after-send](../stories/frontend/composer-error-state-after-send/intake.md) | [50](frontend/50-story-composer-error-state-after-send.md) | To do |
| 51 | BUG-15 | ticket-management | [localized-category-names](../stories/ticket-management/localized-category-names/intake.md) | [51](ticket-management/51-story-localized-category-names.md) | To do |
| 52 | BUG-16 | frontend | [rtl-bidi-isolation](../stories/frontend/rtl-bidi-isolation/intake.md) | [52](frontend/52-story-rtl-bidi-isolation-and-date-formats.md) | To do |
| 53 | BUG-17 | communication-channels | [live-chat-opening-message](../stories/communication-channels/live-chat-opening-message/intake.md) | [53](communication-channels/53-story-live-chat-opening-message.md) | To do |
| 54 | BUG-18 | security-and-administration | [readable-audit-log](../stories/security-and-administration/readable-audit-log/intake.md) | [54](security-and-administration/54-story-readable-audit-log.md) | To do |
| 55 | BUG-19 | sla-and-automation | [sla-deadline-display](../stories/sla-and-automation/sla-deadline-display/intake.md) | [55](sla-and-automation/55-story-sla-deadline-display.md) | To do |
| 56 | BUG-20 | frontend | [ui-polish-qa-batch](../stories/frontend/ui-polish-qa-batch/intake.md) | [56](frontend/56-story-ui-polish-qa-batch.md) | To do |
| 57 | BUG-21 | platform | [validation-message-quality](../stories/platform/validation-message-quality/intake.md) | [57](platform/57-story-validation-message-quality.md) | To do |

## QA round 2 fixes (58–62)

From the round 2 manual QA pass ([report](../qa/2026-10-01-manual-qa-report-round2.md), findings N2–N6). These were fixed straight away, so the plans are as-built. N1 (portal self-registration blocked while email is unconfigured) is part of story 49.

| NN | Id | Feature | Intake | Plan | Status |
|----|----|---------|--------|------|--------|
| 58 | BUG-22 | frontend | [quick-reply-ticket-context](../stories/frontend/quick-reply-ticket-context/intake.md) | [58](frontend/58-story-quick-reply-ticket-context.md) | Done (web `48d8533`) |
| 59 | BUG-23 | security-and-administration | [user-data-access-warning](../stories/security-and-administration/user-data-access-warning/intake.md) | [59](security-and-administration/59-story-user-data-access-warning.md) | Done (api `613e607`, web `6652669`) |
| 60 | BUG-24 | knowledge-base | [article-feedback-limit](../stories/knowledge-base/article-feedback-limit/intake.md) | [60](knowledge-base/60-story-article-feedback-limit.md) | Done (api `93d98b1`, web `79460bd`) |
| 61 | BUG-25 | platform | [upload-response-uploader-name](../stories/platform/upload-response-uploader-name/intake.md) | [61](platform/61-story-upload-response-uploader-name.md) | Done (api `91ec854`) |
| 62 | BUG-26 | frontend | [task-past-date-warning](../stories/frontend/task-past-date-warning/intake.md) | [62](frontend/62-story-task-past-date-warning.md) | Done (web `b91dd22`) |

Backend plans 20–36 are **as-built**: the code was implemented on `customer-support-crm-api` `develop` (commits `c3c7815`, `65c74a3`…`678ea67` and follow-ups) before the plans were written. Each plan cites the commit its line numbers refer to and lists where the code differs from the feature spec.

Feature → plan coverage (backend + frontend) is in [../features/README.md](../features/README.md#plans-and-stories).
