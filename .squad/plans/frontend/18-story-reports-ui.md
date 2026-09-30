# Story 18 — Reports UI (Story: FE-11)

## Prerequisites

- Story 08 completed: [08-story-core-platform-shell.md](08-story-core-platform-shell.md) (`ApiService.download`, `saveBlob`, i18n, permissions, page header, state components).
- Story 09 completed: [09-story-authentication-staff-and-portal.md](09-story-authentication-staff-and-portal.md) (staff session, bearer token on downloads).
- Conventions: [00-overview.md](00-overview.md). This story owns `src/app/features/reports/**` and `public/i18n/reports/{en,ar}.json` only.

---

## Story Goal

1. Staff with `reports.view` open `/reports` and switch between five tabs: management dashboard, ticket volume, SLA performance, agent performance, customer satisfaction.
2. A shared filter bar (date range, default last 30 days; group by day/week/month where the report has a time series; branch and department) drives every tab and is kept while switching tabs.
3. Every report renders at least one Chart.js chart and a data table for the chosen range; the dashboard shows KPI cards.
4. Users with `reports.export` download the CSV of the current tab with the same filters (authenticated download).
5. Everything is translated (en/ar) and RTL-safe, including chart legends, tooltips and axes.

Not in scope: scheduled reports, PDF export, per-agent/category filters (the API does not accept them), backend changes.

---

## Context — Read These Files First

1. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Reports/ReportSlices.cs`
   - line 20: `ReportQuery(From?, To?, BranchId?, DepartmentId?, GroupBy?)` — the only filters of every report.
   - lines 27–41: defaults (last 30 days, `day`), validation: `from < to`, range ≤ 366 days, `groupBy ∈ day|week|month` (`OUT_OF_RANGE`).
   - lines 85–110: `GET /reports/ticket-volume | sla-performance | agent-performance | customer-satisfaction | management-dashboard` (`reports.view`).
   - lines 112–146: `GET /reports/export/{ticket-volume|sla-performance|agent-performance|customer-satisfaction}.csv` (`reports.export`). No CSV for the dashboard.
2. `customer-support-crm-api/src/CustomerSupportCrm.Contracts/Reports/ReportContracts.cs` (lines 3–77) — `TicketVolumeReport`, `SlaPerformanceReport`, `AgentPerformanceReport`, `CustomerSatisfactionReport`, `ManagementDashboard`, `CountByKey`, `VolumePoint`, `SlaBreakdownRow`, `RatingCount`, `SatisfactionPoint`, `SatisfactionComment`.
3. `customer-support-crm-api/src/CustomerSupportCrm.Infrastructure/Reporting/PostgresReportingQueries.cs`
   - line 272: percentages are already 0–100 (one decimal).
   - lines 69, 168, 224: missing keys are returned as `'-'` (no department / no category); status, priority and channel keys are enum names with a `null` label.
   - lines 144–180: the dashboard ignores `from/to/groupBy` (fixed 30/14 days, "today"); only branch/department apply.
4. `customer-support-crm-api/src/CustomerSupportCrm.Domain/Tickets/TicketEnums.cs` — `TicketStatus`, `TicketPriority`, `TicketChannel` values (translated under `reports.enums.*`).
5. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Branches/BranchEndpoints.cs` (lines 140–146) and `Contracts/Organization/OrganizationContracts.cs` (lines 25–34) — `GET /branches` returns `BranchResponse[]` with nested `departments`, readable by any staff member (department picker is derived from it).
6. Web core: `src/app/core/http/api.service.ts` (`get`, `download` → `{ blob, fileName }`), `src/app/shared/file-utils.ts` (`saveBlob`), `src/app/core/localization/translation.service.ts` (`language`, `isRtl`, `t`), `translation.resolver.ts`, `src/app/core/guards/auth.guards.ts` (line 42 `requirePermission`), `src/app/core/permissions/has-permission.directive.ts`, `src/app/shared/page-header.component.ts`, `src/app/shared/state.components.ts`, `src/app/core/localization/localized-date.pipe.ts`.

---

## Frontend Tasks

### 1 — Models, API and filter state (`src/app/features/reports/`)

- `reports.models.ts` — interfaces mirroring the contracts above plus `BranchOption`/`DepartmentOption`; `ReportKind` (`ticket-volume | sla-performance | agent-performance | customer-satisfaction`).
- `reports.api.ts` (`providedIn: 'root'`) — one method per report (`get<T>('/reports/…', { params })`), `branches()` (`/branches`), `exportCsv(kind, params)` → `download('/reports/export/<kind>.csv', { params })`.
- `report-filter.store.ts` — `ReportFilterStore` provided on the feature route (shared by all tabs): `value` signal (`from`, `to` as local dates, `groupBy`, `branchId`, `departmentId`), `rangeError` computed (inverted range or > 366 days), `params` computed (ISO `from` = start of the first day, `to` = start of the day after the last day, empty filters dropped; `null` when invalid so no request is sent), `update()`, `reset()`.
- `report-loader.ts` — `loadReport(params, fetch)` helper: `toObservable` + `switchMap` (cancels stale requests), returns `data`/`loading`/`error` signals and `reload()`; used in field initializers (injection context).
- `report-format.ts` — `ReportFormatter` (root service) for numbers, percentages, minutes (`1h 25m` / `1 س 25 د`), ratings, period labels by `groupBy` (Latin digits in both languages, like `LocalizedDatePipe`) and key labels (`'-'` → "none", enum keys → `reports.enums.*`).

### 2 — Reusable components

- `report-chart.component.ts` — `<app-report-chart type="line|bar|doughnut" [labels] [series] [horizontal] [stacked] [percent] [height] [ariaLabel]>`; registers `registerables` once; creates the chart in `afterRenderEffect`, updates in place when only data/options change, recreates on type change, destroys on `DestroyRef`. Colors resolved from `--mat-sys-primary`, `--mat-sys-tertiary` (resolved through a hidden probe element so `light-dark()` values become `rgb()`), then a fixed fallback palette. Re-renders on language change (labels come from computed translations) and on `prefers-color-scheme` change. RTL: `rtl`/`textDirection` for legend and tooltip, category axis reversed and value axis on the right.
- `report-table.component.ts` — generic `mat-table` with `columns: { key, header (translation key), value(row), numeric? }[]` and `rows`; empty state when no rows.
- `report-filter-bar.component.ts` — date range (`mat-date-range-input`, native date adapter provided locally), optional group-by select, branch select, department select (filtered by branch), reset button, projected actions (export); bound to `ReportFilterStore`. Range errors shown inline.
- `report-export-button.component.ts` — `<app-report-export-button kind="…">` wrapped by `*appHasPermission="'reports.export'"` at the call site; disabled while downloading or when the range is invalid; `saveBlob(blob, fileName)`.
- `reports.scss` — shared page styles (KPI grid, chart/table cards), logical properties only.

### 3 — Pages (child routes of `/reports`)

- `reports-shell.component.ts` — page header + `mat-tab-nav-bar` (`dashboard`, `ticket-volume`, `sla`, `agents`, `satisfaction`) + `<router-outlet>`.
- `management-dashboard.page.ts` — filter bar (branch/department only, note that ranges are fixed), 10 KPI cards, charts + tables: last 14 days (line, created/resolved), backlog by department (bar), top categories 30 days (doughnut).
- `ticket-volume.page.ts` — totals, series chart (line) + table, by status (doughnut), by priority (bar), by channel (bar), by category (horizontal bar), each with a table; CSV export.
- `sla-performance.page.ts` — KPIs, compliance by priority (grouped bar, %) + table, by department (grouped bar, %) + table; CSV export.
- `agent-performance.page.ts` — assigned vs resolved per agent (horizontal grouped bar) + SLA compliance per agent (bar, %) + full table; CSV export.
- `customer-satisfaction.page.ts` — KPIs (average, responses, positive share), trend (line) + table, rating distribution (bar) + table, recent comments list (links to `/tickets/:id`); CSV export.
- `reports.routes.ts` — replaces the placeholder: `REPORTS_ROUTES` root route with `canActivate: [requirePermission(Permissions.reportsView)]`, `resolve: { i18n: translationResolver('reports') }`, `providers: [ReportFilterStore]`, shell component and lazy child pages; `''` → `dashboard`, unknown → `dashboard`.

### 4 — Translations

- `public/i18n/reports/en.json` and `ar.json` — identical key sets: `title`, `tabs`, `filters`, `kpi`, `charts`, `columns`, `enums` (status/priority/channel), `units`, `export`, `comments`, `common`.

---

## Edge Cases & Failure Modes

- Range longer than 366 days or end before start → inline error, no request, export disabled (API would return `OUT_OF_RANGE`).
- Empty date inputs while picking → keep the last valid range until both ends are chosen.
- Switching branch clears a department that belongs to another branch.
- Null metrics (no SLA tickets, no ratings) → "—" in KPIs/tables, `null` points skipped in charts.
- Request failure → global snackbar + `<app-error-state (retry)>`; stale responses cancelled by `switchMap` when filters change quickly.
- User without `reports.export` → export buttons are not rendered (API still enforces 403).
- Language switch → chart labels, legend and axis direction update without reload; charts are destroyed on navigation.

## Test Plan

Out of scope (build-level verification only). Manual smoke: each tab with default range, change range/group by/branch, export each CSV, switch to Arabic and check RTL charts and tables.

## Verification Steps

1. **Frontend builds:** `cd customer-support-crm-web && npx ng build` with no errors or warnings in `src/app/features/reports/**`.

## Done Criteria

- [ ] Each report renders a chart and a table for the chosen range; the dashboard shows KPI cards.
- [ ] Filters (range, group by, branch, department) are shared across tabs and sent to the API.
- [ ] CSV export downloads with auth for users with `reports.export` only.
- [ ] en/ar translations with identical keys; RTL layout and charts.
