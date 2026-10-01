# Story intake

- Folder: `.squad/stories/reports-and-management/reports-and-management-dashboards/intake.md`

---

## Feature

- **Feature name (display):** Reports & Management (feature 09)
- **Feature slug (folder under `plans/`):** `reports-and-management`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `RP-01`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `reports-and-management`

---

## Title

```
Reports and management dashboards (ticket, SLA, agent, CSAT, KPIs, CSV export)
```

---

## Description

```
As built in customer-support-crm-api 2de665a. Code in Application/Features/Reports/ReportSlices.cs,
Contracts/Reports/ReportContracts.cs, Application/Abstractions/Reporting/IReportingQueries.cs,
Infrastructure/Reporting/PostgresReportingQueries.cs, Infrastructure/Persistence/SqlRunner.cs.

Endpoints (/api/v1/reports, reports.view):
- GET /ticket-volume          totals, created/resolved series, by status / priority / channel / category
- GET /sla-performance        compliance % (first response, resolution), breaches, averages,
                              by priority and by department
- GET /agent-performance      per agent: assigned, resolved, open now, avg first-response and
                              resolution minutes, SLA compliance %, avg CSAT, CSAT responses
- GET /customer-satisfaction  average, responses, positive share (rating >= 4), 1..5 distribution,
                              trend, latest 20 comments
- GET /management-dashboard   open / unassigned / escalated / at-risk, created and resolved today,
                              30-day SLA / CSAT / averages, backlog by department, top 5 categories,
                              last 14 days series

Exports (/api/v1/reports/export, reports.view + reports.export), CSV with UTF-8 BOM:
- GET /ticket-volume.csv, /sla-performance.csv, /agent-performance.csv, /customer-satisfaction.csv

Filters (query string, all reports): from, to (default last 30 days, max 366 days, from < to),
branchId, departmentId, groupBy = day | week | month (organization time zone).
Invalid range / groupBy -> 400 VALIDATION_ERROR (OUT_OF_RANGE on "from").

Rules:
- All aggregation in PostgreSQL (hand-written SQL via SqlRunner), never loading ticket rows.
- Every query applies the caller's branch/department AccessScope and excludes soft-deleted tickets.
- CSV fields are quoted and protected against formula injection.
```

---

## Acceptance criteria

```
- [ ] Reports are query slices using projections and database-side aggregation (no loading large datasets into memory).
- [ ] Common filters: date range, branch, department, agent, category, priority, channel.
- [ ] Data scope respects the viewer's permissions and organization context.
- [ ] Export to Excel (and PDF where needed).
- [ ] Dashboard KPIs load within acceptable time on production-size data (indexes / materialized views if required).
- [ ] Permissions Reports.View, Reports.Export, Reports.Management.View enforced.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** ticket-management (NN 25–26), sla-and-automation (NN 28: SLA due/breach columns), customer-portal (NN 30–31: CSAT rating/comment), platform (NN 20–21: `AccessScope`, organization time zone).
- **Depends on code areas or other stories:** `IAccessScopeProvider`, `Organization.TimeZone`, `tickets` table columns, `ApiResults`, `ErrorCodes.OutOfRange`.

## Extra notes (optional)

- "Charts are accessible and support RTL" is a frontend criterion: `../../../plans/frontend/18-story-reports-ui.md` (FE-11).

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10, PostgreSQL (Npgsql).

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- Scheduled / emailed reports, custom report builder.
