# Story intake

- Folder: `.squad/stories/sla-and-automation/sla-deadline-display/intake.md`

---

## Feature

- **Feature name (display):** SLA & Automation
- **Feature slug (folder under `plans/`):** `sla-and-automation`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-19`
- **Work item type:** `Bug`
- **Status:** `To do`
- **Assignee:** ``
- **Labels:** `backend`, `frontend`, `bug`, `sla`, `dashboard`, `reports`

---

## Title

```
Show the SLA deadline that drives the SLA state, and count breached apart from at risk
```

---

## Description

```
Manual QA (2026-10-01, finding L1): on the agent dashboard "SLA at risk" card a
ticket shows the pill "SLA breached" next to "in 15 hours". The first response
deadline has passed (slaState = breached), but the time shown is the
resolution deadline. The /tickets "SLA / due" column does the same: the pill
says "Breached" and the date is the resolution due date.

Causes:
- Both views always print resolutionDueAt
  (features/dashboard/dashboard-ticket-list.component.ts:39-43,
   features/tickets/ticket-list.page.html:155-156).
- TicketListItemResponse has slaState, firstResponseDueAt (null once
  answered) and resolutionDueAt, but not which target set the state or
  firstRespondedAt, so the UI cannot tell "first response overdue" from
  "first response was late, already answered".
- "At risk" counts mix breached and warned tickets: agent dashboard
  MyAtRisk and the SlaAtRisk list (AgentWorkspaceSlices.cs:44), and the
  management dashboard AtRiskTickets (PostgresReportingQueries.cs:151-152).
- Found while investigating: a first-response warning is never cleared when
  the agent replies (Ticket.RecordMessage only sets FirstRespondedAt), and
  SlaState / the at-risk filters read FirstResponseWarnedAt without checking
  that the first response is still pending. An answered ticket stays
  "At risk" until resolution.

Fix:
- API: compute the deadline that drives the state (SlaTarget + SlaDueAt) and
  return FirstRespondedAt in TicketListItemResponse.
- A first-response warning counts only while the first response is pending.
- Split the counts: MyBreached + MyAtRisk (warning only) on the agent
  dashboard, BreachedTickets + AtRiskTickets on the management dashboard.
- The dashboard list stays one list, renamed "SLA at risk or breached",
  breached first, then by the driving deadline.
- Web: one shared component prints the right deadline text
  ("First response overdue · was due 7 hours ago", "Resolution due in 15 hours",
  "First response was late"), used by the dashboard lists and the ticket list.
```

---

## Acceptance criteria

```
- [ ] A ticket whose first response is overdue shows "First response overdue" with the first response deadline, on the dashboard and in the ticket list.
- [ ] A ticket answered late shows "First response was late" and its resolution deadline.
- [ ] A ticket at risk or on track shows the next pending deadline (first response if pending, otherwise resolution).
- [ ] An answered ticket is no longer "At risk" only because of its old first-response warning.
- [ ] The agent dashboard shows separate KPIs for breached and at-risk tickets; the management dashboard shows separate "SLA breached" and "SLA at risk" KPIs.
- [ ] The dashboard list is titled "SLA at risk or breached" and lists breached tickets first.
- [ ] All new texts exist in English and Arabic.
- [ ] `dotnet build` passes with zero warnings; `ng build` passes with zero errors and zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** none. BUG-15 (story 51) also adds a field to `TicketListItemResponse`; the two can land in either order.
- **Depends on code areas or other stories:** stories 25 (tickets), 27 (agent workspace), 28 (SLA engine), 34 (reports); frontend stories 11, 13, 18.
- **Source:** `.squad/qa/2026-10-01-manual-qa-report.md`, finding L1.

## Technical hints (optional)

- Repo roots: `customer-support-crm-api/` (branch `develop`, .NET 10) and `customer-support-crm-web/` (branch `main`, Angular).
- SLA state: `TicketQueries.SlaState` (`Features/Tickets/Common/TicketQueries.cs:48-53`); SLA evaluation: `Ticket.EvaluateSla` / `Check` (`Domain/Tickets/Ticket.cs:360-434`).

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- No e2e tests.
- The meaning of "breached": a breach flag stays set for the ticket's life (also after a late reply); this story does not change that.
- The ticket list "SLA / due" sort (still sorts by resolution due date) and an SLA filter in the ticket list UI.
- Linking the KPI tiles to a pre-filtered ticket list.
- The ticket details SLA block in the side panel (it already shows both targets separately).
