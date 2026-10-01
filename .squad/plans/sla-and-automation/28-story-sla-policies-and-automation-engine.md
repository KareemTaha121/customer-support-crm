# Story 28 — SLA policies, assignment rules, escalation rules and the SLA evaluation job (Story: SL-01)

> As-built plan: written after implementation. The SLA/automation **domain** (`Domain/Sla/*`), EF configuration (`SlaConfiguration.cs`) and `IApplicationDbContext.Automation.cs` were committed in `customer-support-crm-api` `57e52f8`; the application layer (`Features/Sla/*`), contracts, DI, the recurring job and the seed in `b81ba28`; the tables in migration `20260930104602_AddSupportOperations` (`678ea67`). None of these files changed after `b81ba28`, so line numbers are the same at `b81ba28` and at `develop` HEAD `2956767` (except `Infrastructure/DependencyInjection.cs` and `DatabaseInitializer.cs`, cited at `2956767`).

## Prerequisites

- Ticket Management completed: [../ticket-management/25-story-ticket-lifecycle.md](../ticket-management/25-story-ticket-lifecycle.md) and [../ticket-management/26-story-ticket-conversation-and-categories.md](../ticket-management/26-story-ticket-conversation-and-categories.md) — `Ticket` SLA fields, `ApplySla`, `EvaluateSla`, SLA domain events, `TicketQueries.CanHandleAsync` / `EligibleAgents` / `FindTrackedAsync`, `TicketSlaEventHandlers`.
- Platform phase 3 (`65c74a3`) — `RecurringRequestService<TRequest>` / `BackgroundJobOptions`, `NotificationSender`, `Organization.TimeZone`, realtime push of new notifications in `ApplicationDbContext.SaveChangesAsync`.
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

1. Admins define SLA policies: first-response and resolution minutes per priority, optional category/department scope, one default policy, optional business hours.
2. New tickets get due dates automatically; changes of priority, category or department recompute them.
3. Assignment rules route new tickets (department, priority) and pick an agent (specific, round-robin, least loaded).
4. A job raises SLA warnings (80 %) and breaches once per clock, and fires time-based escalation rules once per ticket.
5. Escalation rules react to SLA warning/breach or waiting time: raise priority, reassign, escalate, notify.

**Deviations from the intake (the code is authoritative):**

| Intake (feature spec 05) | As built in `b81ba28` |
|---|---|
| `SlaPolicy`, `SlaTarget`, `SlaEscalationRule` | `SlaPolicy` + `SlaTargetTime` (table `sla_targets`; `SlaTarget` is the FirstResponse/Resolution enum); `EscalationRule` + `EscalationRun`; `AssignmentRule` |
| `BusinessHours` / `Holiday` entities, `Features/Sla/BusinessHours` | Business hours are fields on the policy (`BusinessHoursOnly`, `WorkDays`, `WorkStart`, `WorkEnd`) in the organization time zone; **no holidays** |
| SLA clock pauses in `PendingCustomer` | **Not built** — due dates are fixed at apply time; no pause |
| Policies by priority/category/**customer tier** | Priority + category + department; no customer tier |
| `SlaApproaching` / `SlaBreached` events | `TicketSlaWarningDomainEvent` / `TicketSlaBreachedDomainEvent` (`Domain/Tickets/TicketDomainEvents.cs` lines 22–24); warning threshold fixed at 0.8 (`EvaluateSlaHandler.WarningThreshold`) |
| Assignment: round-robin, load-balanced, skill/category, per department; fallback to department queue | `SpecificAgent`, `RoundRobin`, `LeastLoaded`, `None` (routing only); category/department/channel/priority/keyword conditions; no match or no candidate → ticket stays unassigned in its department (the queue is implicit) |
| `Features/Automation/` and `Features/Notifications/` slices | All in `Features/Sla/` (`SlaAdministration.cs`, `SlaEngine.cs`); notifications list/read live in `Features/Notifications` (phase 3) |
| Notification flow via `INotificationService` → provider adapter (in-app, email, SMS, WhatsApp) | `NotificationSender` writes in-app `Notification` rows, pushed over SignalR after commit; **no email/SMS/WhatsApp to staff** |
| Notification preferences per user and channel | **Not built** |
| Automation logged in ticket history | Indirect: priority/assignee/escalation/SLA events write history rows; escalation reason is `"Automatic: {rule name}"`; notifications and rule runs are not history rows |
| Escalate to supervisor / next level | `NotifyManagers` = active users with `tickets.assign` in the ticket's scope; `ReassignToAgentId` for reassignment |

**Not in scope:** agent dashboard, tasks/reminders, quick replies (same commit, feature 04). **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Domain/Sla/SlaPolicy.cs` — lines 11–175. Properties (31–60), `Create` (62), `Update` validation (64–109: name ≤ 150, business hours need ≥ 1 day and end > start, one target per priority, resolution ≥ first response > 0 → `INVALID_SLA_POLICY`), `ClearDefault` (111), `TargetFor` (113), `AddWorkingMinutes` (116–151, bounded 800-day walk), `SlaTargetTime` (154–175).
2. `src/CustomerSupportCrm.Domain/Sla/AutomationRules.cs` — `AssignmentStrategy` (7–14), `AssignmentRule` (20–140: `Update` 74–111, `Matches` 113–124, `PickRoundRobin` 127–139 with `LastAssignedAgentId` cursor), `EscalationTrigger` (142–152), `EscalationRule` (155–261: `Update` 211–250, time triggers need `afterMinutes > 0`, `Matches` 252–260), `EscalationRun` (264–283).
3. `src/CustomerSupportCrm.Domain/Tickets/Ticket.cs` — SLA fields (79–99), `ApplySla` (350–357: keeps `FirstResponseDueAt` once responded, resets warnings), `EvaluateSla` (360–376), `Check` (396–434: breach once, warn once at threshold of elapsed/total).
4. `src/CustomerSupportCrm.Application/Features/Tickets/Common/TicketQueries.cs` — `CanHandleAsync` (227–233), `EligibleAgents(db, branchId, departmentId, permission)` (236–240), `SlaState` (48–53).
5. `src/CustomerSupportCrm.Infrastructure/BackgroundJobs/RecurringRequestService.cs` — lines 22–59 (fresh scope per tick, errors logged, `BackgroundJobs:Enabled`), `AddRecurringRequest` (63–65).
6. `src/CustomerSupportCrm.Application/Features/Notifications/NotificationSlices.cs` — `NotificationSender` (24–45); `Infrastructure/Persistence/ApplicationDbContext.cs` `SaveChangesAsync` (44–65) pushes `RealtimeEvents.NotificationCreated` after commit.

---

## Backend Tasks

### 1 — Domain and persistence (`57e52f8`)

- Files: `Domain/Sla/SlaPolicy.cs`, `Domain/Sla/AutomationRules.cs` (see Context).
- `src/CustomerSupportCrm.Application/Abstractions/Persistence/IApplicationDbContext.Automation.cs` lines 9–15: `SlaPolicies`, `AssignmentRules`, `EscalationRules`, `EscalationRuns` (plus `AgentTasks`, `QuickReplies` for feature 04).
- `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/SlaConfiguration.cs`: `SlaPolicyConfiguration` (11–27: `work_days integer[]`, category/department FKs SetNull, targets cascade, field access, `xmin`), `SlaTargetTimeConfiguration` (29–37: key (PolicyId, Priority)), `AssignmentRuleConfiguration` (39–55: enums as strings, index on Order, `xmin`), `EscalationRuleConfiguration` (57–72: `notify_user_ids uuid[]`, `xmin`), `EscalationRunConfiguration` (74–85: unique (RuleId, TicketId), cascades).
- Migration `20260930104602_AddSupportOperations.cs`: `assignment_rules` (line 65), `escalation_rules` (137), `sla_policies` (654), `sla_targets` (771), `escalation_runs` (867); `tickets.sla_policy_id` (709) is a plain nullable column — **no FK** to `sla_policies`.

### 2 — Contracts

File: `src/CustomerSupportCrm.Contracts/Sla/SlaContracts.cs` (1–96): `SlaTargetDto` (3), `SlaPolicyRequest` / `SlaPolicyResponse` (7–32), `AssignmentRuleRequest` / `AssignmentRuleResponse` (35–63), `EscalationRuleRequest` / `EscalationRuleResponse` (67–96). Enums travel as names.

### 3 — Administration slices (`Features/Sla/SlaAdministration.cs`)

- `AutomationErrors` (21–25): `SLA_POLICY_NOT_FOUND`, `RULE_NOT_FOUND`. `SlaMapping` (27–48: `HH:mm` format/parse, `ParseEnum`).
- SLA policies: `ListSlaPoliciesHandler` (52–58, default first), `SaveSlaPolicyValidator` (62–78: priority names case-sensitive, first response 1–86 400 min, resolution 1–525 600 min, `HH:mm` regex), `SaveSlaPolicyHandler` (80–123: defaults work days Sun–Thu `[0..4]`, 08:00–17:00; clears `IsDefault` on others 111–117; audit `sla_policies.created|updated`), `DeleteSlaPolicyHandler` (128–138, audit `sla_policies.deleted`; the doc comment at 127 says the reference is cleared, but nothing clears `tickets.sla_policy_id` — it keeps the deleted id and `sla.policyName` becomes null; due dates stay).
- Assignment rules: list (142–154, ordered, with agent name), mapping (156–173), validator (177–188), save (190–227, audit `assignment_rules.*`).
- Escalation rules: list (231–237, by name), mapping (239–256), validator (260–271: `afterMinutes` 1–43 200), save (273–309, audit `escalation_rules.*`).
- `DeleteAutomationRuleHandler` (312–334, `Kind` = `assignment` | `escalation`, audit `{kind}_rules.deleted`).
- `SlaAdministrationEndpoints` (338–405): `/sla-policies` — GET (any authenticated user), POST (201) / PUT / DELETE with `sla.manage`; `/automation/assignment-rules` and `/automation/escalation-rules` groups with `.RequireAuthorization(Permissions.AutomationManage)` on the group (365, 385). Permissions `sla.manage` / `automation.manage` in `Domain/Roles/Permissions.cs` lines 28–29 (in `ManagerDefaults`).

### 4 — Engine services (`Features/Sla/SlaEngine.cs`)

Registered scoped in `Application/DependencyInjection.cs` lines 47–49.

- `SlaCalculator.ApplyAsync` (16–49): active policies with targets; candidates matching category/department, excluding unscoped non-default policies (25); score category = 2, department = 1 (26); no policy or no target for the priority → `ApplySla(null, null, null)`; start = `ticket.CreatedAt`; organization time zone via `TimeZoneInfo.TryFindSystemTimeZoneById`, UTC fallback (44–48).
- `AssignmentEngine.ApplyAsync` (52–123): rules by `Order`, `CreatedAt`; first match; `SetDepartmentId` → `TransferTo` if the department is active (65–72); `SetPriority`; skip agent selection if already assigned or `None`; `SpecificAgent` only if `CanHandleAsync`; `RoundRobin` over eligible `tickets.update` holders ordered by id; `LeastLoaded` counts open tickets per agent (107–122).
- `EscalationExecutor.ExecuteAsync` (126–165): skip inactive tickets; raise priority only upwards (137–140); reassign if eligible; `Escalate("Automatic: {rule.Name}", automatic: true)` unless Resolved; notify listed users + assignee + managers (`EligibleAgents(..., tickets.assign)`) with `ticket.escalated`.

### 5 — Event handlers (`SlaEngine.cs`)

- `TicketRoutingHandler` (169–183) on `TicketCreatedDomainEvent`: assignment rules first, then SLA (so a rule-set priority/department drives the policy).
- `SlaRecalculationHandler` (186–208) on priority / category / transferred events: re-applies SLA for active tickets that are not newly added.
- `SlaEscalationHandler` (210–234) on SLA warning / breach events: runs every matching active rule for that trigger and target.
- The ticket feature's `TicketSlaEventHandlers` (`Features/Tickets/TicketEventHandlers.cs` 177–204) write the `sla_warning` / `sla_breached` history rows and notify the assignee.

### 6 — Background evaluation

- `EvaluateSlaCommand` / `EvaluateSlaHandler` (`SlaEngine.cs` 242–295): `WarningThreshold = 0.8`, `BatchSize = 500`; loads active tickets with an open clock ordered by `ResolutionDueAt` and calls `EvaluateSla`; then for each active `Unassigned` / `NoAgentReply` rule finds tickets past `AfterMinutes` without an `EscalationRun` (273–279), executes the rule and records an `EscalationRun`; one `SaveChangesAsync` (dispatches the SLA events above).
- Registration: `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` line 103 `services.AddRecurringRequest<EvaluateSlaCommand>(TimeSpan.FromMinutes(1))`.

### 7 — Seed

`src/CustomerSupportCrm.Infrastructure/Persistence/Seed/DatabaseInitializer.cs` lines 65–87: when no policy exists, a default **Standard** policy (24×7) — Urgent 30 min / 4 h, High 1 h / 8 h, Medium 4 h / 24 h, Low 8 h / 72 h.

### 8 — Localization

`Messages.resx` lines 192–195: `SLA_POLICY_NOT_FOUND`, `RULE_NOT_FOUND`. `Messages.ar.resx` lines 246–258 add `INVALID_SLA_POLICY`, `INVALID_ASSIGNMENT_RULE`, `INVALID_ESCALATION_RULE` (`0936711`); in English the domain codes fall back to the exception message.

---

## Edge Cases & Failure Modes

- **No default policy and no scoped match** — ticket has no due dates (`SlaState` `none`).
- **Policy for a category but no target for the ticket's priority** — no due dates (the default policy is not tried as a fallback).
- **Two default policies** — prevented by the save handler; deleting the default leaves none.
- **Business hours with no working day / end ≤ start** — 422 `INVALID_SLA_POLICY`.
- **Unknown time zone id** — UTC.
- **Priority raised by escalation** — `SlaRecalculationHandler` recomputes due dates from `CreatedAt` and clears warnings; breach flags stay.
- **Ticket already responded** — `ApplySla` keeps the original `FirstResponseDueAt`; `EvaluateSla` skips the first-response clock.
- **Job restarts / double ticks** — breach/warn flags and `EscalationRun` (unique index) keep actions single; SLA-triggered rules fire once because the event is raised once.
- **Round-robin cursor agent left the pool** — `IndexOf` = −1, starts at the first candidate.
- **`SpecificAgent` / `ReassignToAgentId` not eligible** — silently skipped.
- **Job failure** — logged by `RecurringRequestService` (`LogJobFailed`), retried next minute; disabled with `BackgroundJobs:Enabled = false`.
- **Concurrent rule edits** — `xmin` → 409 `CONFLICT`.

---

## Test Plan

Test projects are **out of scope** (`tests/` must not be modified). No tests in the repo at `2956767` cover SLA policies, assignment/escalation rules or the evaluation job.

---

## Verification Steps

1. **Backend builds:** `dotnet build` in `customer-support-crm-api/` — 0 warnings, 0 errors.
2. **Seed:** on a fresh database `GET /api/v1/sla-policies` → one `Standard` policy, `isDefault: true`, four targets.
3. **Create policy:** `curl -X POST …/api/v1/sla-policies -H "Authorization: Bearer $TOKEN" -d '{"name":"Billing BH","isActive":true,"isDefault":false,"categoryId":"<Billing>","businessHoursOnly":true,"workDays":[0,1,2,3,4],"workStart":"08:00","workEnd":"17:00","targets":[{"priority":"High","firstResponseMinutes":60,"resolutionMinutes":480}]}'` → 201; `resolutionMinutes` < `firstResponseMinutes` → 422 `INVALID_SLA_POLICY`.
4. **Apply:** create a High ticket in Billing → `sla.policyName` "Billing BH", due dates inside working hours; change priority to Low → due dates recomputed (Billing BH has no Low target → none).
5. **Assignment:** `POST …/automation/assignment-rules {"name":"RR support","isActive":true,"order":1,"strategy":"RoundRobin"}` → new tickets alternate between eligible agents; history shows `assignee`.
6. **Escalation:** rule `{"trigger":"Unassigned","afterMinutes":1,"escalateTicket":true,"notifyManagers":true}`; leave a ticket unassigned > 1 min → `status: Escalated`, history `escalated` "Level 1: Automatic: …", managers get `ticket.escalated`; it does not fire again.
7. **Breach:** policy with `firstResponseMinutes: 1` → after the next tick `sla.state` `warning` then `breached`, assignee notified, `GET /tickets?sla=breached` lists it.
8. **Permissions:** Agent token on `POST /sla-policies` or `GET /automation/assignment-rules` → 403.

---

## Done Criteria

- [x] SLA policies with per-priority targets, category/department scope, single default, business hours; CRUD with `sla.manage`.
- [x] Ticket tracks `FirstResponseDueAt`, `ResolutionDueAt`, `FirstRespondedAt`, `ResolvedAt`, breach flags and warning timestamps.
- [ ] SLA clock respects holidays and pauses in `PendingCustomer` — not built (business hours only).
- [x] Background job (every minute) raises SLA warning (80 %) and breach events once.
- [x] Ordered, configurable assignment rules (specific, round-robin, least loaded, routing only); unmatched tickets stay in their department.
- [x] Escalation rules on SLA warning/breach and time-based triggers: raise priority, reassign, escalate, notify; idempotent.
- [ ] `INotificationService` with email/SMS/WhatsApp adapters for staff — not built; in-app notifications only.
- [ ] Notification preferences per user and channel — not built.
- [ ] Automation actions logged in ticket history — partially (through the resulting ticket events only).
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding.**
