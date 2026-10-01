# Story 55 — Show the SLA deadline that drives the SLA state, and count breached apart from at risk (Bug: BUG-19)

> Fix plan, not yet implemented. Paths and line numbers refer to customer-support-crm-api `dca992c` (`develop`) and customer-support-crm-web `06c817a` (`main`).
> Intake: [../../stories/sla-and-automation/sla-deadline-display/intake.md](../../stories/sla-and-automation/sla-deadline-display/intake.md)

## Prerequisites

- Story 28 — [28-story-sla-policies-and-automation-engine.md](28-story-sla-policies-and-automation-engine.md): `Ticket.EvaluateSla`, the SLA monitor (every minute, `Infrastructure/DependencyInjection.cs:103`).
- Story 25 — [../ticket-management/25-story-ticket-lifecycle.md](../ticket-management/25-story-ticket-lifecycle.md): `TicketQueries.SlaState`, `ProjectListAsync`, the `sla` list filter.
- Story 27 — [../agent-dashboard/27-story-agent-workspace.md](../agent-dashboard/27-story-agent-workspace.md): `GetAgentDashboardHandler`.
- Story 34 — [../reports-and-management/34-story-reports-and-management-dashboards.md](../reports-and-management/34-story-reports-and-management-dashboards.md): `ManagementDashboardAsync`.
- Frontend stories 11, 13 and 18 — [../frontend/11-story-tickets-ui.md](../frontend/11-story-tickets-ui.md), [../frontend/13-story-agent-dashboard-ui.md](../frontend/13-story-agent-dashboard-ui.md), [../frontend/18-story-reports-ui.md](../frontend/18-story-reports-ui.md).
- Story 51 — [../ticket-management/51-story-localized-category-names.md](../ticket-management/51-story-localized-category-names.md) also adds a field to `TicketListItemResponse`. The two stories do not depend on each other.
- Source: manual QA report `.squad/qa/2026-10-01-manual-qa-report.md`, finding L1.
- Backend paths are relative to **`customer-support-crm-api/`**, frontend paths to **`customer-support-crm-web/`**.

---

## Story Goal

**How the state is computed today.** The SLA monitor calls `Ticket.EvaluateSla` (`Domain/Tickets/Ticket.cs:360–376`). It checks the first response only while it is pending, and the resolution while it has a due date. `Check` (396–434) sets `…Breached = true` once `now >= due`, otherwise `…WarnedAt` once the warning threshold of elapsed time is reached. Breach flags are never cleared.

`TicketQueries.SlaState` (`Features/Tickets/Common/TicketQueries.cs:48–53`) then returns:

- `breached` if either flag is set (also for resolved tickets);
- `none` if the ticket is inactive;
- `warning` if either `…WarnedAt` is set;
- `ok` if a due date exists.

**What the UI prints.** Both the dashboard card (`features/dashboard/dashboard-ticket-list.component.ts:36–43`) and the ticket list (`features/tickets/ticket-list.page.html:152–159`) print the state pill and then always `resolutionDueAt`. A ticket whose first response is 7 h overdue but whose resolution is due in 15 h reads "SLA breached · in 15 hours".

The list item cannot do better. `TicketListItemResponse` (`Contracts/Tickets/TicketContracts.cs:35–57`) has `FirstResponseDueAt`, which is nulled once answered (`TicketQueries.cs:108`), and `ResolutionDueAt`. It has no `FirstRespondedAt` and does not say which target set the state, although the projection already reads `FirstRespondedAt` (81).

**Stale first-response warning (found while investigating).** `Ticket.RecordMessage` sets only `FirstRespondedAt` on the first public agent reply (`Ticket.cs:324–327`). `FirstResponseWarnedAt` stays set. `SlaState` and all "at risk" queries read it without checking that the first response is pending. So a ticket warned before its first reply stays "At risk" until it is resolved or its SLA is re-applied (`ApplySla`, 350–357).

**Counts.** "At risk" means warned *or* breached everywhere:

- agent dashboard `atRisk` (`Features/Dashboard/AgentWorkspaceSlices.cs:44`), used for `MyAtRisk` (50) and the `SlaAtRisk` list (59, sorted by `ResolutionDueAt`);
- the `sla=at_risk` list filter (`Features/Tickets/TicketQuerySlices.cs:24`, 142–143);
- management dashboard `AtRiskTickets` (`Infrastructure/Reporting/PostgresReportingQueries.cs:151–152`).

| | Before | After |
|---|---|---|
| Deadline shown | Always the resolution due date | The deadline that drives the state (`SlaTarget` + `SlaDueAt` from the API) |
| First response overdue | "SLA breached · in 15 hours" | "SLA breached · First response overdue · was due 7 hours ago" |
| Answered late, resolution pending | "Breached · <resolution date>" | "Breached · First response was late · Resolution due in 15 hours" |
| Warned, then answered | Stays "At risk" | Back to "On track" (unless the resolution is warned) |
| Agent KPIs | `myAtRisk` = warned + breached | `myBreached` and `myAtRisk` (warned, not breached) |
| Management KPIs | `atRiskTickets` = warned + breached | `breachedTickets` and `atRiskTickets` (warned, not breached) |
| Dashboard list | "SLA at risk", sorted by resolution due | "SLA at risk or breached", breached first, then by `SlaDueAt` |

**Choice: one list, split counts.** The QA asked to separate breached from at risk. The two numbers answer different questions ("already late" vs "about to be late"), and counts are cheap, so the KPIs are split on both dashboards. The dashboard **list** stays one list of the ten most urgent tickets: both kinds need the agent's attention, and two cards would double the space for a list that is usually short. It is renamed so its title no longer claims that breached tickets are only "at risk", and each row now says which deadline is overdue or next. The `sla=at_risk` API filter keeps its documented meaning (warning or breached) and now uses the same predicates.

**Deviation from the intake:** none. The QA report did not mention the stale first-response warning. It is fixed here because it produces the same kind of contradictory badge, and the fix is a single predicate.

Note that `src/app/shared/**` and `public/i18n/core/*` are edited: the deadline text is used by two features (dashboard and tickets). `frontend/00-overview.md` reserves those folders for platform stories, but duplicating the component in both features would split one rule in two.

---

## Context — Read These Files First

Backend:

1. `src/CustomerSupportCrm.Domain/Tickets/Ticket.cs`: `RecordMessage` (317–348, first reply at 324–327), `ApplySla` (350–357), `EvaluateSla` (360–376), `Check` (396–434).
2. `src/CustomerSupportCrm.Domain/Tickets/TicketEnums.cs`: `SlaTarget { FirstResponse, Resolution }` (42–46).
3. `src/CustomerSupportCrm.Application/Features/Tickets/Common/TicketQueries.cs`: `SlaState` (48–53), `ProjectListAsync` (55–115; SLA fields 75–81, mapping 107–109), `GetResponseAsync` (138–139).
4. `src/CustomerSupportCrm.Contracts/Tickets/TicketContracts.cs`: `TicketListItemResponse` (34–57).
5. `src/CustomerSupportCrm.Application/Features/Tickets/TicketQuerySlices.cs`: `Sla` doc (24), validator (56), filter switch (138–145), `dueAt` sort (168).
6. `src/CustomerSupportCrm.Application/Features/Dashboard/AgentWorkspaceSlices.cs`: `GetAgentDashboardHandler` (29–75).
7. `src/CustomerSupportCrm.Contracts/Dashboard/DashboardContracts.cs`: `AgentDashboardCounts` (5–14), `AgentDashboardResponse` (33–39).
8. `src/CustomerSupportCrm.Infrastructure/Reporting/PostgresReportingQueries.cs`: `ManagementDashboardAsync` (144–196), the counts query at 146–160.
9. `src/CustomerSupportCrm.Contracts/Reports/ReportContracts.cs`: `ManagementDashboard` (64–77).

Frontend:

10. `src/app/features/dashboard/dashboard-ticket-list.component.ts`: template 36–43. It is used by three cards in `dashboard.page.html`: my tickets (56), SLA (101–106) and escalations (108–113).
11. `src/app/features/dashboard/dashboard.page.ts`: `KPIS` (32–42). `dashboard.models.ts`: `AgentDashboardCounts` (2–12), `DashboardTicket` (14–31).
12. `src/app/features/tickets/ticket-list.page.html`: the `dueAt` column (148–162). `tickets.models.ts`: `TicketListItem` (15–38), `slaTone` (250).
13. `src/app/features/reports/management-dashboard.page.ts`: KPI tiles (80–91, at-risk at 84). `reports.models.ts`: `ManagementDashboard` (108–122).
14. `src/app/core/localization/localized-date.pipe.ts`: `'relative'` (41–58) gives "in 15 hours" / "7 hours ago" (ar: "خلال 15 ساعة" / "قبل 7 ساعات").
15. i18n: `public/i18n/dashboard/{en,ar}.json` (`kpi` 14–24, `lists` 25–37, `sla` 53–58), `public/i18n/reports/{en,ar}.json` (`kpi.atRisk` 38), `public/i18n/core/{en,ar}.json`.

---

## Backend Tasks

### 1 — One definition of "breached" and "at risk"

In `TicketQueries` (add `using System.Linq.Expressions;`):

```csharp
/// <summary>A breach flag is set (it stays set for the ticket's life).</summary>
public static readonly Expression<Func<Ticket, bool>> SlaBreached =
    t => t.FirstResponseBreached || t.ResolutionBreached;

/// <summary>Not breached, and a still-pending target has been warned.</summary>
public static readonly Expression<Func<Ticket, bool>> SlaWarning =
    t => !t.FirstResponseBreached && !t.ResolutionBreached
        && ((t.FirstRespondedAt == null && t.FirstResponseWarnedAt != null) || t.ResolutionWarnedAt != null);

/// <summary>Breached or warning.</summary>
public static readonly Expression<Func<Ticket, bool>> SlaBreachedOrWarning =
    t => t.FirstResponseBreached || t.ResolutionBreached
        || (t.FirstRespondedAt == null && t.FirstResponseWarnedAt != null) || t.ResolutionWarnedAt != null;
```

In `SlaState` (51), count a first-response warning only while it is pending. Both callers already pass `firstDue` as "pending only" (lines 107 and 138):

```csharp
: (firstWarned is not null && firstDue is not null) || resolutionWarned is not null ? "warning"
```

### 2 — The deadline that drives the state

Add next to `SlaState`:

```csharp
/// <summary>The SLA target that sets <see cref="SlaState"/> and its deadline; null when inactive or no SLA.</summary>
public static (string? Target, DateTimeOffset? DueAt) SlaFocus(
    bool active, bool firstBreached, bool resolutionBreached, DateTimeOffset? firstWarned, DateTimeOffset? resolutionWarned,
    DateTimeOffset? firstRespondedAt, DateTimeOffset? firstDue, DateTimeOffset? resolutionDue)
{
    var first = (nameof(SlaTarget.FirstResponse), firstDue);
    var resolution = (nameof(SlaTarget.Resolution), resolutionDue);
    var firstPending = firstRespondedAt is null && firstDue is not null;

    if (!active) return (null, null);
    if (firstPending && firstBreached) return first;              // first response overdue
    if (resolutionBreached && resolutionDue is not null) return resolution; // resolution overdue
    if (firstBreached && firstDue is not null) return first;      // answered late
    if (firstPending && firstWarned is not null) return first;    // first response at risk
    if (resolutionWarned is not null && resolutionDue is not null) return resolution;
    if (firstPending) return first;                               // on track: next pending deadline
    return resolutionDue is not null ? resolution : (null, null);
}
```

Pass `firstDue` **raw** (`t.FirstResponseDueAt`), not the nulled value, so that the "answered late" branch keeps its deadline. (Adjust the tuple syntax to the project's style. The project uses braces on every `if`.)

The order mirrors `SlaState`: the first three branches are the `breached` cases, the next two the `warning` cases, and the last two `ok`. A first response that is overdue comes before an overdue resolution, because the reply is the action the agent can take now.

### 3 — List item contract

`TicketListItemResponse`: after `ResolutionDueAt` add

```csharp
DateTimeOffset? FirstRespondedAt,
/// FirstResponse or Resolution: the target that sets SlaState; null when inactive or without SLA.
string? SlaTarget,
DateTimeOffset? SlaDueAt,
```

Update the XML doc on the record (34). In `ProjectListAsync`, compute `var focus = SlaFocus(t.Status.IsActive(), t.FirstResponseBreached, t.ResolutionBreached, t.FirstResponseWarnedAt, t.ResolutionWarnedAt, t.FirstRespondedAt, t.FirstResponseDueAt, t.ResolutionDueAt);` and pass `t.FirstRespondedAt, focus.Target, focus.DueAt`. The lambda at 91 becomes a block lambda. `ProjectListAsync` is the only constructor call site; the agent dashboard lists use it too.

### 4 — Ticket list filter

`TicketQuerySlices` (140–143): `"breached" => query.Where(TicketQueries.SlaBreached)`, `"warning" => query.Where(TicketQueries.SlaWarning)`, and `"at_risk" =>` active statuses `.Where(TicketQueries.SlaBreachedOrWarning)`. The doc at line 24 stays correct ("warning or breached").

### 5 — Agent dashboard

In `GetAgentDashboardHandler`:

- Replace `atRisk` (44) with `var attention = active.Where(TicketQueries.SlaBreachedOrWarning);`.
- Counts: `MyBreached = await mine.Where(TicketQueries.SlaBreached).CountAsync(ct)` and `MyAtRisk = await mine.Where(TicketQueries.SlaWarning).CountAsync(ct)`.
- List (59), breached first, then by the driving deadline (the same choice as `SlaFocus` for pending first responses):

```csharp
attention
    .OrderBy(t => t.FirstResponseBreached || t.ResolutionBreached ? 0 : 1)
    .ThenBy(t => t.FirstRespondedAt == null && t.FirstResponseDueAt != null ? t.FirstResponseDueAt : t.ResolutionDueAt)
    .Take(ListSize)
```

`AgentDashboardCounts`: add `int MyBreached` right after `MyAtRisk`, and document `MyAtRisk` as "warned, not breached". There is a single constructor call site (47–56).

### 6 — Management dashboard

In the counts query (`PostgresReportingQueries.cs:151–152`), replace the at-risk column with two:

```sql
count(*) FILTER (WHERE t.status NOT IN ('Resolved', 'Closed')
     AND NOT (t.first_response_breached OR t.resolution_breached)
     AND ((t.first_responded_at IS NULL AND t.first_response_warned_at IS NOT NULL) OR t.resolution_warned_at IS NOT NULL)),
count(*) FILTER (WHERE t.status NOT IN ('Resolved', 'Closed') AND (t.first_response_breached OR t.resolution_breached)),
```

Shift the reader (159) to `AtRisk: r.GetInt64(3), Breached: r.GetInt64(4), CreatedToday: r.GetInt64(5), ResolvedToday: r.GetInt64(6)`.

`ManagementDashboard`: add `long BreachedTickets` right after `AtRiskTickets`, and pass `current.Breached` after `current.AtRisk` (186). This is the only call site.

**No changes to:** domain, migrations, resx, `TicketSlaResponse` (the details panel already shows both targets), the `dueAt` sort, or docs (`docs/endpoints.md` does not list response fields).

---

## Frontend Tasks

### 1 — Models

- `TicketListItem` (`tickets.models.ts`) and `DashboardTicket` (`dashboard.models.ts`): add `firstRespondedAt: string | null`, `slaTarget: 'FirstResponse' | 'Resolution' | null`, `slaDueAt: string | null`.
- `AgentDashboardCounts`: add `myBreached: number`.
- `ManagementDashboard` (`reports.models.ts`): add `breachedTickets: number`.

### 2 — Shared deadline component

Create `src/app/shared/sla-due.component.ts` (`app-sla-due`, standalone, OnPush, imports `TranslatePipe` and `LocalizedDatePipe`):

```ts
export interface SlaDueSource {
  slaTarget: 'FirstResponse' | 'Resolution' | null;
  slaDueAt: string | null;
  firstRespondedAt: string | null;
  resolutionDueAt: string | null;
}

/** The SLA deadline that drives the ticket's SLA state, e.g. "First response overdue · was due 7 hours ago". */
export class SlaDueComponent {
  readonly ticket = input.required<SlaDueSource>();
  readonly format = input<'relative' | 'short'>('relative');

  protected readonly lines = computed<{ key: string; at: string | null }[]>(() => {
    const t = this.ticket();
    if (!t.slaTarget || !t.slaDueAt) {
      return [];
    }
    const overdue = (iso: string) => new Date(iso).getTime() <= Date.now();
    if (t.slaTarget === 'FirstResponse' && t.firstRespondedAt) {
      const rest = t.resolutionDueAt ? [{ key: overdue(t.resolutionDueAt) ? 'core.sla.resolutionOverdue' : 'core.sla.resolutionDue', at: t.resolutionDueAt }] : [];
      return [{ key: 'core.sla.firstResponseLate', at: null }, ...rest];
    }
    const target = t.slaTarget === 'FirstResponse' ? 'firstResponse' : 'resolution';
    return [{ key: `core.sla.${target}${overdue(t.slaDueAt) ? 'Overdue' : 'Due'}`, at: t.slaDueAt }];
  });
}
```

The template renders each line as `<small>` (with a `schedule` icon in relative mode) and the text `{{ line.key | t: { when: (line.at | localDate: format()) } }}`. "Overdue" is decided by the clock rather than the breach flag, because the monitor runs once a minute. `now` is read when the inputs change: each dashboard refresh and each list load. That is enough for a list, and the side panel keeps its own live view.

### 3 — Use it

- `dashboard-ticket-list.component.ts`: replace lines 39–43 with `<app-sla-due [ticket]="ticket" />`. Keep the state pill (36–38). This changes all three dashboard cards, which is intended: "My tickets" had the same problem.
- `ticket-list.page.html`: replace 155–156 with `<app-sla-due [ticket]="row" format="short" />` and keep the `—` for `slaState === 'none'` (157–159). Add `SlaDueComponent` to both components' `imports`.
- `dashboard.page.ts` `KPIS`: insert `{ key: 'myBreached', icon: 'alarm', tone: 'danger', link: '/tickets' }` before `myAtRisk`, and change `myAtRisk`'s tone to `'warning'`.
- `management-dashboard.page.ts`: after the at-risk tile (84) add `{ icon: 'alarm', label: 'reports.kpi.breached', value: f.number(d.breachedTickets), tone: d.breachedTickets ? 'danger' : undefined }`.

### 4 — i18n (en / ar)

`public/i18n/core/{en,ar}.json`, a new top-level `sla` block (`{{when}}` is a relative or short date):

| Key | en | ar |
|---|---|---|
| `core.sla.firstResponseDue` | First response due {{when}} | الرد الأول مستحق {{when}} |
| `core.sla.firstResponseOverdue` | First response overdue · was due {{when}} | الرد الأول متأخر · كان مستحقًا {{when}} |
| `core.sla.firstResponseLate` | First response was late | تأخر الرد الأول |
| `core.sla.resolutionDue` | Resolution due {{when}} | الحل مستحق {{when}} |
| `core.sla.resolutionOverdue` | Resolution overdue · was due {{when}} | الحل متأخر · كان مستحقًا {{when}} |

`public/i18n/dashboard/{en,ar}.json`:

| Key | en | ar |
|---|---|---|
| `dashboard.kpi.myBreached` (new) | My tickets with SLA breached | تذاكري المتجاوزة لاتفاقية الخدمة |
| `dashboard.kpi.myAtRisk` (unchanged text, now warning only) | My tickets at SLA risk | تذاكري المعرضة لخرق اتفاقية الخدمة |
| `dashboard.lists.slaAtRisk` (changed) | SLA at risk or breached | معرضة لخرق اتفاقية الخدمة أو متجاوزة لها |
| `dashboard.lists.slaAtRiskEmpty` (changed) | No tickets are at risk of breaching SLA or past an SLA deadline. | لا توجد تذاكر معرضة لخرق اتفاقية مستوى الخدمة أو متجاوزة لها. |

`public/i18n/reports/{en,ar}.json`: `reports.kpi.breached` (new): "SLA breached" / "متجاوزة لاتفاقية الخدمة". `reports.kpi.atRisk` is unchanged.

---

## Test Plan

Test projects are **out of scope**: `tests/` must not be modified. No e2e.

---

## Verification Steps

1. **Backend build:** `dotnet build src/CustomerSupportCrm.Api/CustomerSupportCrm.Api.csproj` gives 0 warnings and 0 errors. Use `-o <temp dir>` if a running API locks `bin/`.
2. **Frontend build:** `cd customer-support-crm-web && npx ng build` gives 0 errors and 0 warnings.
3. **Set up:** an SLA policy with a short first-response target (e.g. 5 min) and a long resolution target (e.g. 24 h). Create ticket A, assign it to yourself, and do not reply. Wait for the breach (the monitor runs every minute).
4. **Overdue first response:** `/dashboard` "SLA at risk or breached" lists A first: "SLA breached · First response overdue · was due N minutes ago". `/tickets` shows "Breached" and "First response overdue · was due <date>". `GET /api/v1/tickets` returns `slaTarget: "FirstResponse"` and `slaDueAt` = `firstResponseDueAt`.
5. **Answered late:** reply publicly to A. The rows show "First response was late" and "Resolution due in ~24 hours". The response has `firstRespondedAt` set, `slaTarget: "FirstResponse"`, and `slaState` still `breached`.
6. **Stale warning:** ticket B, warned on first response (wait past the threshold, before the due time), then answered. `slaState` returns to `ok`, B leaves the dashboard list, and `myAtRisk` drops.
7. **On track:** a new ticket C shows "First response due in …". After a reply it shows "Resolution due in …".
8. **KPIs:** the agent dashboard shows "My tickets with SLA breached" = 1 (A) and "My tickets at SLA risk" counts only warned tickets. `/reports` management dashboard shows "SLA breached" and "SLA at risk" separately; their sum equals the old at-risk figure minus stale warnings.
9. **Arabic:** switch to Arabic and repeat steps 4–8 visually. The texts read e.g. "الرد الأول متأخر · كان مستحقًا قبل 7 ساعات".

---

## Done Criteria

- [ ] `TicketListItemResponse` carries `firstRespondedAt`, `slaTarget` and `slaDueAt`, computed by `TicketQueries.SlaFocus`.
- [ ] The dashboard cards and the ticket list show the deadline that drives the state (overdue, late, or next pending).
- [ ] A first-response warning no longer counts after the first reply (`SlaState`, list filters, dashboard, management report).
- [ ] Breached and at-risk are counted separately on both dashboards; the list is renamed and sorted breached first.
- [ ] New and changed texts exist in en and ar.
- [ ] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [ ] `dotnet build` and `ng build` pass with zero warnings.

**STOP HERE. Report to the user.**
