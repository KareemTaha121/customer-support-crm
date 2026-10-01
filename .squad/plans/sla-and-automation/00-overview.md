# SLA & Automation

Feature spec: [../../features/05-sla-and-automation.md](../../features/05-sla-and-automation.md).
Repo: `customer-support-crm-api`. Out of scope for every story: Docker, deployment, CI/CD, and files under `tests/`.
Matching frontend plan: [../frontend/14-story-sla-and-automation-admin-ui.md](../frontend/14-story-sla-and-automation-admin-ui.md) (FE-07); SLA badges on tickets are in [../frontend/11-story-tickets-ui.md](../frontend/11-story-tickets-ui.md).

| NN | Tracker id | Plan file | Title | Depends on | Status |
|----|-----------|-----------|-------|-----------|--------|
| 28 | SL-01 | [28-story-sla-policies-and-automation-engine.md](28-story-sla-policies-and-automation-engine.md) | SLA policies, assignment rules, escalation rules and the SLA evaluation job | Ticket Management 25–26 | Done |
| 39 | BUG-03 | [39-story-ticket-sla-policy-foreign-key.md](39-story-ticket-sla-policy-foreign-key.md) | Clear ticket SLA policy references when a policy is deleted | 28 | Done (`3433f13`) |
| 55 | BUG-19 | [55-story-sla-deadline-display.md](55-story-sla-deadline-display.md) | Show the deadline that drives the SLA state; count breached apart from at risk; clear stale warnings (QA L1) | 28, 27, 34 | To do |

The plan is an **as-built** plan written after implementation, from the real code.

Commits:

- `57e52f8` (feat: add ticket management) — SLA/automation domain (`Domain/Sla/SlaPolicy.cs`, `Domain/Sla/AutomationRules.cs`), `SlaConfiguration.cs`, `IApplicationDbContext.Automation.cs`, and the SLA fields/events on `Ticket`.
- `b81ba28` (feat: add SLA & automation and agent workspace) — `Features/Sla/SlaAdministration.cs`, `Features/Sla/SlaEngine.cs`, `Contracts/Sla/SlaContracts.cs`, DI, the 1-minute `EvaluateSlaCommand` job, the seeded **Standard** policy. The dashboard/tasks/quick-replies part of this commit belongs to the agent-dashboard feature.
- `678ea67` — migration `20260930104602_AddSupportOperations` creates the SLA/automation tables.
- `0936711` — Arabic messages for the SLA/automation error codes.

No later commit touches `Features/Sla`, `Domain/Sla`, `Contracts/Sla` or `SlaConfiguration.cs`.

Known gaps and drift (spec vs as built):

- **No holidays and no SLA pause in `PendingCustomer`**; business hours are fields on each policy, in the organization time zone.
- **No customer-tier targeting**; policies match category and/or department, else the default.
- **Warning threshold is hard-coded** at 80 % (`EvaluateSlaHandler.WarningThreshold`).
- **Notifications are in-app only** (`NotificationSender` + SignalR push); no `INotificationService` with email/SMS/WhatsApp adapters for staff, and **no notification preferences**.
- **No separate `Features/Automation` / `Features/Notifications` slices** as in the spec; everything is in `Features/Sla` (notification list/read endpoints already existed from phase 3).
- **Automation is logged only through the ticket events it causes** (priority, assignee, escalated, SLA rows); there is no history row naming the assignment rule.
- ~~**Deleting an SLA policy leaves a dangling `tickets.sla_policy_id`** (no FK; the handler comment says it is cleared, it is not).~~ — fixed by story 39 (`3433f13`); [intake](../../stories/sla-and-automation/ticket-sla-policy-foreign-key/intake.md)
- **Reopen does not reset SLA** due dates or breach flags.
- **No tests** for SLA or automation in `tests/`.
