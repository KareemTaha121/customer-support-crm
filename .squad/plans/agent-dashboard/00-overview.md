# Agent Dashboard

Feature spec: [../../features/04-agent-dashboard.md](../../features/04-agent-dashboard.md).
Repo: `customer-support-crm-api`. Out of scope for every story: Docker, deployment, CI/CD, and files under `tests/`.

| NN | Tracker id | Plan file | Title | Depends on | Status |
|----|-----------|-----------|-------|-----------|--------|
| 27 | AD-01 | [27-story-agent-workspace.md](27-story-agent-workspace.md) | Agent workspace: dashboard, tasks & reminders, quick replies | Tickets (02), Customers (23) | Done |
| 38 | BUG-02 | [38-story-task-scope-and-reference-validation.md](38-story-task-scope-and-reference-validation.md) | Tasks: enforce scope on ticket/customer filters and validate task references | 27 | Done (`7058b62`) |

Implemented in `customer-support-crm-api` commit `b81ba28` (feat: add SLA & automation (feature 05) and agent workspace (feature 04)) without a plan; only `Features/Dashboard/AgentWorkspaceSlices.cs`, `Contracts/Dashboard/DashboardContracts.cs` and the reminder job registration belong to this feature (the SLA part is feature 05). The `AgentTask` / `QuickReply` entities, EF configuration and DbSets came earlier in `57e52f8` (feature 02); the tables are created by the migration in `678ea67`; messages in `0936711`. This is an **as-built** plan; line numbers refer to `b81ba28`. Nothing in `Features/Dashboard` has changed since.

Frontend: [../frontend/13-story-agent-dashboard-ui.md](../frontend/13-story-agent-dashboard-ui.md).

Known gaps and drift:

- **Collaboration** — ticket watchers/followers and handover are not built. @mentions exist only on ticket messages (feature 02) and send `ticket.mentioned` notifications.
- **Unread messages widget** counts unread notifications, not unread ticket/chat messages.
- **Near-real-time counters** — the dashboard is pull-only; only notifications are pushed (`notificationCreated` on `/hubs/staff`).
- **Quick replies** — "shared" means global, not per department; `CategoryId` has no entity or validation; no grouped localized variants.
- ~~**Tasks** — ticket/customer links are not validated (unknown id → 500 from the foreign key); listing tasks by `ticketId`/`customerId` is not scope-checked and returns every assignee's tasks.~~ — fixed by story 38 (`7058b62`); [intake](../../stories/agent-dashboard/task-scope-and-reference-validation/intake.md)
- **Permissions** — tasks and quick-reply endpoints have no endpoint permission (any authenticated staff user); rules use `tickets.assign` / `quickreplies.manage` inside the handlers.
- **No tests** cover the dashboard, tasks, reminders or quick replies.
