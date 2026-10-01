# Story 34 — Reports and management dashboards (Story: RP-01)

> As-built plan: written after implementation in `customer-support-crm-api` commit `2de665a` (feat: add reports and management dashboards (feature 09)). Paths and line numbers refer to `2de665a`; the reporting files are unchanged at `develop` HEAD `2956767` (no follow-up commits touch `Features/Reports`, `Infrastructure/Reporting`, `Contracts/Reports` or `SqlRunner.cs`). Files from other commits name their commit.

## Prerequisites

- platform (NN 20–21, `65c74a3`): `AccessScope`, `IAccessScopeProvider`, `Organization.TimeZone`, `IEndpoint` staff group.
- ticket-management (NN 25–26): `tickets` columns `status`, `priority`, `channel`, `category_id`, `branch_id`, `department_id`, `assigned_agent_id`, `created_at`, `resolved_at`, `first_responded_at`, `is_deleted`.
- sla-and-automation (NN 28): `sla_policy_id`, `first_response_due_at`, `resolution_due_at`, `first_response_breached`, `resolution_breached`, `*_warned_at`.
- customer-portal (NN 30–31): `satisfaction_rating`, `satisfaction_comment`, `satisfaction_submitted_at`.
- Permissions `reports.view` / `reports.export` already in the catalog (`src/CustomerSupportCrm.Domain/Roles/Permissions.cs` lines 35–36, `65c74a3`), granted to `ManagerDefaults` (line 80).
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

Supervisors and managers get five read-only reports and four CSV exports over a date range, sliced by branch/department and grouped by day/week/month in the organization time zone — always computed in PostgreSQL and always limited to the caller's data scope.

**Deviations from the intake (the code is authoritative):**

| Feature spec (`09-reports-and-management.md`) | As built in `2de665a` |
|---|---|
| Slices `TicketVolume/`, `SlaPerformance/`, `AgentPerformance/`, `CustomerSatisfaction/`, `ManagementDashboard/`, `Export/` | One file `Features/Reports/ReportSlices.cs` with minimal-API handlers calling `IReportingQueries` directly (no MediatR request per report) |
| Projections (EF) | Hand-written PostgreSQL SQL through `SqlRunner` (`date_trunc … AT TIME ZONE`, `FILTER` aggregates) |
| Filters: date range, branch, department, **agent, category, priority, channel** | Only `from`, `to`, `branchId`, `departmentId`, `groupBy`; category / priority / channel are breakdowns, agent is a per-row report, not filters |
| Export to Excel and PDF | CSV only (UTF-8 BOM so Excel opens Arabic correctly); no `.xlsx`, no PDF; no management-dashboard export |
| `Reports.Management.View` | Not in the permission catalog; management dashboard requires `reports.view` |
| Indexes / materialized views for dashboard KPIs | No new indexes or views; relies on ticket indexes from migration `20260930104602_AddSupportOperations` (`678ea67`: `ix_tickets_created_at`, `ix_tickets_branch_id_department_id_status`, `ix_tickets_assigned_agent_id_status`); nothing on `resolved_at` or `satisfaction_submitted_at` |
| Validation via FluentValidation | `ReportFilterFactory` throws `ValidationException` itself (single `OUT_OF_RANGE` failure on `From`) |
| — | Also in `2de665a`: `PostgresSequenceGenerator` and `PostgresKnowledgeSearch` moved onto the shared `SqlRunner` |

**Not in scope:** chart rendering / RTL (frontend FE-11), scheduled reports. **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Application/Abstractions/Reporting/IReportingQueries.cs` — `ReportFilter(From, To, BranchId, DepartmentId, AccessScope Scope, TimeZone, GroupBy = "day")` (6–14); `IReportingQueries` (16–31), five methods.
2. `src/CustomerSupportCrm.Application/Features/Reports/ReportSlices.cs` — `ReportQuery` (20), `ReportFilterFactory` (23–42), `Csv` (44–79), `ReportEndpoints` (81–148).
3. `src/CustomerSupportCrm.Contracts/Reports/ReportContracts.cs` — lines 1–77 (all response records).
4. `src/CustomerSupportCrm.Infrastructure/Reporting/PostgresReportingQueries.cs` — `Scope` SQL (16–21), `PeriodOf` (24), reports (26–196), helpers (198–272).
5. `src/CustomerSupportCrm.Infrastructure/Persistence/SqlRunner.cs` — `QueryAsync` (14–25), `ScalarAsync` (27–32), `CreateCommandAsync` (34–53, enlists in the current EF transaction, line 46), `ReaderExtensions` (56–66).
6. `src/CustomerSupportCrm.Application/Common/Authorization/AccessScope.cs` (`65c74a3`) — `AccessScope(AllBranches, BranchIds, DepartmentIds)` (14), `IAccessScopeProvider` (30).
7. `src/CustomerSupportCrm.Application/DependencyInjection.cs` line 47 — `AddScoped<ReportFilterFactory>()`. `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` lines 74 and 76 — `AddScoped<SqlRunner>()`, `AddScoped<IReportingQueries, PostgresReportingQueries>()`.
8. `src/CustomerSupportCrm.Contracts/Common/ErrorCodes.cs` line 29 — `OutOfRange = "OUT_OF_RANGE"`.
9. `src/CustomerSupportCrm.Infrastructure/Persistence/Migrations/20260930104602_AddSupportOperations.cs` (`678ea67`) — satisfaction columns 723–725, ticket indexes 1199–1243.
10. `docs/endpoints.md` lines 166–171 (at `c8e9251` / HEAD; the catalog was added in `fdd93cd`).

---

## Backend Tasks

### 1 — Contracts (`Contracts/Reports/ReportContracts.cs`)

```csharp
public sealed record CountByKey(string Key, string? Label, long Count);
public sealed record VolumePoint(DateTimeOffset Period, long Created, long Resolved);
public sealed record TicketVolumeReport(From, To, GroupBy, TotalCreated, TotalResolved, Series, ByStatus, ByPriority, ByChannel, ByCategory);
public sealed record SlaBreakdownRow(string Key, string? Label, long Tickets, double? FirstResponseCompliance, double? ResolutionCompliance, long Breached);
public sealed record SlaPerformanceReport(From, To, TicketsWithSla, FirstResponseCompliance, ResolutionCompliance,
    FirstResponseBreached, ResolutionBreached, AverageFirstResponseMinutes, AverageResolutionMinutes, ByPriority, ByDepartment);
public sealed record AgentPerformanceRow(AgentId, AgentName, Assigned, Resolved, OpenNow, AverageFirstResponseMinutes,
    AverageResolutionMinutes, SlaCompliance, AverageSatisfaction, SatisfactionResponses);
public sealed record AgentPerformanceReport(From, To, Agents);
public sealed record RatingCount(int Rating, long Count);
public sealed record SatisfactionPoint(DateTimeOffset Period, double? Average, long Responses);
public sealed record SatisfactionComment(Guid TicketId, string TicketNumber, int Rating, string Comment, DateTimeOffset SubmittedAt);
public sealed record CustomerSatisfactionReport(From, To, Average, Responses, PositiveShare, Distribution, Trend, RecentComments);
public sealed record ManagementDashboard(OpenTickets, UnassignedTickets, EscalatedTickets, AtRiskTickets, CreatedToday, ResolvedToday,
    SlaCompliance30d, Satisfaction30d, AverageFirstResponseMinutes30d, AverageResolutionMinutes30d,
    BacklogByDepartment, TopCategories30d, Last14Days);
```

Percentages are 0–100 with one decimal (`Percent`), averages two decimals (`Round`), `null` when there is no data.

### 2 — Filter (`ReportSlices.cs` 19–42)

`ReportQuery(From?, To?, BranchId?, DepartmentId?, GroupBy?)` bound with `[AsParameters]`. `ReportFilterFactory.CreateAsync`: `to = To ?? now`, `from = From ?? to − 30 days`, `groupBy = GroupBy ?? "day"`; `from >= to`, range > `MaxRange` (366 days) or `groupBy` not `day|week|month` → `ValidationException` (`From`, `OUT_OF_RANGE`); time zone = first organization's `TimeZone` or `UTC`; scope from `IAccessScopeProvider`.

### 3 — SQL implementation (`PostgresReportingQueries`)

- `Scope` fragment on alias `t`: `NOT t.is_deleted AND (@all OR t.branch_id = ANY(@branches) OR t.department_id = ANY(@departments))` plus optional `@branch` / `@department`. A `branchId` outside the caller's scope therefore returns empty results, not an error.
- `PeriodOf(col)` = `date_trunc(@unit, col AT TIME ZONE @tz) AT TIME ZONE @tz`.
- **TicketVolume** (26–42): `SeriesAsync` (created and resolved CTEs `FULL JOIN`ed by period); breakdowns by `status`, `priority`, `channel`, `category_id` (label = category name) on tickets created in range.
- **SlaPerformance** (44–70): tickets with `sla_policy_id` created in range; compliance = share not breached among tickets whose target was met or breached; breach counts; average first-response / resolution minutes over all tickets in range; rows by priority and by department.
- **AgentPerformance** (72–110): `JOIN users` on `assigned_agent_id`; assigned (created in range), resolved in range, open now, average times, resolution SLA compliance, CSAT; agents with no activity omitted; ordered by resolved desc, then name.
- **CustomerSatisfaction** (112–142): rated in range by `satisfaction_submitted_at`; average, count, positive share (rating ≥ 4); distribution padded to ratings 1–5; trend by period; latest 20 non-null comments.
- **ManagementDashboard** (144–196): current counters (open, unassigned, `Escalated`, at risk = breached or warned, created/resolved since local midnight); 30-day SLA (resolution compliance) and CSAT by re-using the methods with `filter with { From = now − 30d, To = now }`; backlog by department (top 10); top 5 categories (30 days); last 14 days daily series. `from`/`to`/`groupBy` do not affect the dashboard.
- `Parameters(filter)` (256–268) builds fresh `NpgsqlParameter`s per command (uuid arrays for scope); all values are bound — SQL text is constant (`#pragma warning disable CA2100` in `SqlRunner` line 43).

### 4 — Endpoints (`ReportEndpoints : IEndpoint`, 81–148)

Group `/reports`, tag `Reports`, `.RequireAuthorization(Permissions.ReportsView)`:

| Route | Name | Returns |
|---|---|---|
| `GET /ticket-volume` | `TicketVolumeReport` | `ApiResponse<TicketVolumeReport>` |
| `GET /sla-performance` | `SlaPerformanceReport` | `ApiResponse<SlaPerformanceReport>` |
| `GET /agent-performance` | `AgentPerformanceReport` | `ApiResponse<AgentPerformanceReport>` |
| `GET /customer-satisfaction` | `CustomerSatisfactionReport` | `ApiResponse<CustomerSatisfactionReport>` |
| `GET /management-dashboard` | `ManagementDashboard` | `ApiResponse<ManagementDashboard>` |

Sub-group `/reports/export` adds `.RequireAuthorization(Permissions.ReportsExport)` (both policies apply):

| Route | Columns |
|---|---|
| `GET /ticket-volume.csv` | `period, created, resolved` (series) |
| `GET /sla-performance.csv` | `department, tickets, first_response_compliance_pct, resolution_compliance_pct, breached` |
| `GET /agent-performance.csv` | `agent, assigned, resolved, open_now, avg_first_response_min, avg_resolution_min, sla_compliance_pct, avg_csat, csat_responses` |
| `GET /customer-satisfaction.csv` | `period, average, responses` (trend) |

`Csv.File` (46–57): UTF-8 BOM, `text/csv; charset=utf-8`, invariant culture, dates `yyyy-MM-dd HH:mm`; `Escape` (67–78) prefixes `'` to non-numeric values starting with `= + - @` and quotes values containing `,`, `"` or newlines.

### 5 — Infrastructure plumbing

- New `SqlRunner` (scoped) runs raw SQL on the DbContext connection and transaction.
- `PostgresSequenceGenerator` now calls `sql.ScalarAsync<long>("SELECT nextval(@name::regclass)", …)`; `PostgresKnowledgeSearch` refactored onto `SqlRunner` (same commit).
- No migration, no domain entity, no resx keys (validation message is inline English; `OUT_OF_RANGE` and `VALIDATION_ERROR` are generic codes).

---

## Edge Cases & Failure Modes

- **No parameters** — last 30 days, grouped by day.
- **`from >= to`, range > 366 days, `groupBy=year`** — 400 `VALIDATION_ERROR`, field `from`, code `OUT_OF_RANGE`.
- **Branch outside scope / unknown id** — empty report (scope AND filter), no 403.
- **No data** — totals 0, averages/percentages `null`, CSAT distribution still lists 1–5 with 0.
- **Unassigned tickets** — excluded from agent performance (inner join).
- **Time zone** — buckets and "today" use `Organization.TimeZone`; a missing organization falls back to `UTC`.
- **Missing permission** — `reports.view` absent → 403 on everything; `reports.export` absent → 403 on CSV only.
- **CSV injection** — agent names or labels starting with `=` are neutralized.
- **Large datasets** — every aggregate is one SQL statement; the dashboard runs about 14 statements sequentially on one connection.

---

## Test Plan

Test projects are **out of scope** (`tests/` must not be modified). No tests were added for this story, and no existing test in `tests/` covers the reports or `SqlRunner` (searched at `2956767`).

---

## Verification Steps

1. **Backend builds:** in `customer-support-crm-api/` run `dotnet build` — 0 warnings, 0 errors.
2. **Volume:** `curl "https://localhost:<port>/api/v1/reports/ticket-volume?from=2026-09-01T00:00:00Z&to=2026-10-01T00:00:00Z&groupBy=week" -H "Authorization: Bearer $TOKEN"` → `series`, `byStatus`, `byChannel`, totals.
3. **Validation:** `…/reports/ticket-volume?groupBy=year` → 400 `OUT_OF_RANGE`; `from` after `to` → 400.
4. **SLA / agents / CSAT:** `GET …/reports/sla-performance`, `/agent-performance`, `/customer-satisfaction` → shapes above; CSAT `distribution` has five entries.
5. **Dashboard:** `GET …/reports/management-dashboard?branchId=<id>` → counters limited to that branch; `last14Days` daily points.
6. **Export:** `curl -OJ …/reports/export/agent-performance.csv -H "Authorization: Bearer $TOKEN"` → CSV starting with a BOM and the header row.
7. **Permissions:** an Agent token (no `reports.view`) → 403; a role with `reports.view` but not `reports.export` → 200 on reports, 403 on `/export/*.csv`.
8. **Scope:** a supervisor scoped to one department sees only that department's numbers.

---

## Done Criteria

- [x] Five report endpoints and four CSV exports exist, use the standard envelope (reports) / file result (CSV), and enforce `reports.view` / `reports.export`.
- [x] All aggregation runs in PostgreSQL; no ticket rows are loaded into memory.
- [x] Date range (max 366 days), branch, department and day/week/month grouping in the organization time zone.
- [x] Every query applies the caller's `AccessScope` and excludes soft-deleted tickets.
- [ ] Filters by agent, category, priority, channel — not built (breakdowns only).
- [ ] Excel (`.xlsx`) and PDF export — not built; CSV only.
- [ ] `Reports.Management.View` permission — not built; dashboard uses `reports.view`.
- [ ] Indexes / materialized views tuned for dashboard KPIs — not added; no performance measurement recorded.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/` by this story.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to the next feature.**
