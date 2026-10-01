# Reports & Management — feature 09

Feature spec: [../../features/09-reports-and-management.md](../../features/09-reports-and-management.md).
Repo: `customer-support-crm-api`. Out of scope for every story: Docker, deployment, CI/CD, and files under `tests/`.

| NN | Tracker id | Plan file | Title | Depends on | Status |
|----|-----------|-----------|-------|-----------|--------|
| 34 | RP-01 | [34-story-reports-and-management-dashboards.md](34-story-reports-and-management-dashboards.md) | Reports and management dashboards (ticket, SLA, agent, CSAT, KPIs, CSV export) | 20–21, 25–26, 28, 30–31 | Done |

Story intake: [`../../stories/reports-and-management/reports-and-management-dashboards/intake.md`](../../stories/reports-and-management/reports-and-management-dashboards/intake.md).
Frontend counterpart: [../frontend/18-story-reports-ui.md](../frontend/18-story-reports-ui.md) (FE-11 — report pages, filter bar, charts, CSV download, management dashboard).

The story was implemented in `customer-support-crm-api` commit `2de665a` (feat: add reports and management dashboards (feature 09)) on `develop`, before this plan was written; the plan is **as-built** and its line numbers refer to `2de665a`. No later commit touches `Features/Reports`, `Infrastructure/Reporting`, `Contracts/Reports` or `Infrastructure/Persistence/SqlRunner.cs` (checked up to `2956767`). The ticket columns and indexes the SQL relies on come from migration `20260930104602_AddSupportOperations` (`678ea67`).

Known gaps and drift:

- **Export is CSV only.** The spec asks for Excel and PDF; neither is built, and the management dashboard has no export.
- **Filters are limited** to date range, branch, department and grouping. Agent, category, priority and channel filters from the spec are not built (they appear only as breakdowns / per-agent rows).
- **No `Reports.Management.View` permission.** The management dashboard is guarded by `reports.view` like the other reports.
- **No reporting indexes or materialized views.** KPIs rely on the general ticket indexes; `resolved_at` and `satisfaction_submitted_at` are not indexed, and no performance check on production-size data was recorded.
- **Slices are not one-folder-per-use-case.** One `ReportSlices.cs` file with minimal-API handlers calling `IReportingQueries`; no MediatR requests or FluentValidation validators.
- **A branch/department filter outside the caller's scope returns an empty report** instead of 403 `OUT_OF_SCOPE`.
- **The dashboard ignores `from` / `to` / `groupBy`** (fixed windows: today, 30 days, 14 days).
- **No automated tests** for reports in `tests/`.
- Accessible, RTL charts are a frontend criterion (FE-11), not covered by this backend story.
