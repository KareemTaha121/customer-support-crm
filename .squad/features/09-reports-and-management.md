# 09 — Reports & Management

> **Source:** AZM Squad Customer Support CRM — Core Features §9
> **Implementation phase:** Phase 11 — Reports
> **Status:** Done (backend + frontend) — backend plans [34](../plans/reports-and-management/00-overview.md), frontend plan [18](../plans/frontend/18-story-reports-ui.md)
> **Build priority:** 14

## Summary

Give supervisors and management clear, reliable numbers on volume, speed, quality and satisfaction — sliced by branch, department, agent, channel and time.

## Scope

| Sub-feature | Description |
|-------------|-------------|
| Ticket reports | Volume, backlog, by status/category/priority/channel, trends. |
| SLA performance | Compliance %, breaches, average first-response and resolution time. |
| Agent performance | Tickets handled, resolution time, reopen rate, CSAT per agent. |
| Customer satisfaction | CSAT scores, trends, comments. |
| Management dashboards | Executive KPI overview across branches/departments. |

## User stories

- As a **supervisor**, I view ticket volume and backlog for my department over a date range.
- As a **supervisor**, I see SLA compliance and which tickets breached.
- As a **manager**, I compare agent performance.
- As a **manager**, I see CSAT trends and read customer comments.
- As a **manager**, I export reports to Excel/PDF.

## Acceptance criteria

- [ ] Reports are query slices using projections and database-side aggregation (no loading large datasets into memory).
- [ ] Common filters: date range, branch, department, agent, category, priority, channel.
- [ ] Data scope respects the viewer's permissions and organization context.
- [ ] Export to Excel (and PDF where needed).
- [ ] Dashboard KPIs load within acceptable time on production-size data (indexes / materialized views if required).
- [ ] Charts are accessible and support RTL.

## Backend slices

```text
Features/Reports/
├── TicketVolume/
├── SlaPerformance/
├── AgentPerformance/
├── CustomerSatisfaction/
├── ManagementDashboard/
└── Export/
```

## Frontend

- `features/reports`: report pages with shared filter bar, charts, tables, export.
- Management dashboard page.

## Dependencies

- Ticket Management (02), SLA (05), Customer Portal feedback (08), Security (10).

## Permissions

`Reports.View`, `Reports.Export`, `Reports.Management.View`
