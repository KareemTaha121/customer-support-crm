# 05 — SLA & Automation

> **Source:** AZM Squad Customer Support CRM — Core Features §5
> **Implementation phase:** Phase 7 — SLA & Automation
> **Status:** Done (backend + frontend) — backend plans [28, 39](../plans/sla-and-automation/00-overview.md), frontend plan [14](../plans/frontend/14-story-sla-and-automation-admin-ui.md)
> **Build priority:** 9–10 (SLA → Notifications)

## Summary

Make sure every ticket is answered and resolved on time: define targets, route tickets automatically, escalate when targets are at risk, and notify the right people.

## Scope

| Sub-feature | Description |
|-------------|-------------|
| Response and resolution targets | SLA policies by priority/category/customer tier, with business hours and holidays. |
| Automatic assignment | Rules: round-robin, load-balanced, skill/category-based, per department. |
| Escalation rules | Escalate on approaching/breached SLA to supervisor/next level. |
| Alerts and notifications | In-app, email, SMS, WhatsApp notifications with user preferences. |

## User stories

- As an **admin**, I define SLA policies with first-response and resolution targets per priority.
- As a **supervisor**, new tickets are automatically assigned to available agents in the right department.
- As a **supervisor**, I am alerted when a ticket is about to breach SLA and it escalates automatically on breach.
- As a **user**, I choose how I receive notifications.

## Acceptance criteria

- [ ] SLA is a domain capability: `SlaPolicy`, `SlaTarget`, `SlaEscalationRule`.
- [ ] Ticket tracks `FirstResponseDueAt`, `ResolutionDueAt`, `FirstRespondedAt`, `ResolvedAt`.
- [ ] SLA clock respects business hours, holidays and pauses in `PendingCustomer`.
- [ ] Background job evaluates SLA and emits `SlaApproaching` / `SlaBreached` events.
- [ ] Assignment rules are configurable and ordered; fallback to department queue.
- [ ] Escalation rules act on events (reassign, raise priority, notify).
- [ ] Notification flow: event → handler → `INotificationService` → provider adapter (no direct provider calls from business handlers).
- [ ] Notification preferences per user and channel.
- [ ] Automation actions are idempotent and logged in ticket history.

## Domain model

```text
SlaPolicy
SlaTarget
SlaEscalationRule
BusinessHours / Holiday
AssignmentRule
Notification
NotificationPreference
```

## Backend slices

```text
Features/Sla/ (Policies CRUD, BusinessHours, EvaluateSla job)
Features/Automation/ (AssignmentRules CRUD, AutoAssign handler, EscalationRules CRUD, Escalate handler)
Features/Notifications/ (List, MarkRead, Preferences, Send)
```

## Frontend

- SLA badges/countdowns on ticket list and details.
- Admin screens: SLA policies, business hours, assignment rules, escalation rules.
- Notification bell/center and preferences page.

## Dependencies

- Ticket Management (02), Communication Channels (03) for outbound notifications, Platform background jobs.
