# 04 — Agent Dashboard

> **Source:** AZM Squad Customer Support CRM — Core Features §4
> **Implementation phase:** Phase 6 — Agent Workspace
> **Status:** Done (backend + frontend) — backend plans [27](../plans/agent-dashboard/00-overview.md), frontend plan [13](../plans/frontend/13-story-agent-dashboard-ui.md)
> **Build priority:** 8

## Summary

The agent's daily workspace: what is assigned to me, what is urgent, who the customer is, what I need to do next, and how I work with my team.

## Scope

| Sub-feature | Description |
|-------------|-------------|
| Assigned tickets | My open tickets, SLA-at-risk, pending escalations, unread messages. |
| Customer information | Customer summary panel alongside the ticket. |
| Tasks and reminders | Personal/ticket-linked tasks with due dates and reminders. |
| Quick replies | Reusable canned responses (personal and shared), with placeholders, AR/EN. |
| Team collaboration | Internal notes, @mentions, ticket watchers/followers, handover. |

## User stories

- As an **agent**, I open the dashboard and see my tickets sorted by urgency/SLA.
- As an **agent**, I see the customer's profile and recent history next to the ticket.
- As an **agent**, I create tasks and get reminded before they are due.
- As an **agent**, I insert a quick reply into a message with customer/ticket placeholders filled.
- As an **agent**, I @mention a colleague on a ticket and they are notified.

## Acceptance criteria

- [ ] Dashboard widgets use dedicated lightweight query projections (`MyTickets`, `MyTasks`, `SlaAtRisk`, `RecentCustomers`, `UnreadMessages`, `PendingEscalations`).
- [ ] Dashboard never loads full ticket aggregates.
- [ ] Tasks: create, update, complete, due date, reminder; optional ticket/customer link.
- [ ] Reminders fire via background job → Notifications.
- [ ] Quick replies: personal vs shared (by department), categories, placeholders, localized variants.
- [ ] Mentions create in-app notifications and appear in the ticket timeline.
- [ ] Widget counters refresh in near real-time.

## Domain model

```text
AgentTask
Reminder
QuickReply
TicketWatcher
Mention
```

## Backend slices

```text
Features/AgentWorkspace/
├── Dashboard/ (MyTickets, SlaAtRisk, RecentCustomers, UnreadMessages, PendingEscalations)
├── Tasks/ (Create, Update, Complete, List)
├── QuickReplies/ (CRUD, Render)
└── Collaboration/ (Mention, Watch, Unwatch)
```

## Frontend

- `features/dashboard`: widget grid, responsive.
- Quick-reply picker inside the ticket composer.
- Tasks panel and reminders.

## Dependencies

- Ticket Management (02), Customer Management (01), SLA (05), Notifications (05).
