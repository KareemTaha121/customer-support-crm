# Story intake

- Folder: `.squad/stories/sla-and-automation/sla-policies-and-automation-engine/intake.md`

---

## Feature

- **Feature name (display):** SLA & Automation
- **Feature slug (folder under `plans/`):** `sla-and-automation`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `SL-01`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `sla`

---

## Title

```
SLA policies, assignment rules, escalation rules and the SLA evaluation job
```

---

## Description

```
Admins define SLA policies and automation rules; the engine applies them to tickets.
Domain in Domain/Sla (SlaPolicy, SlaTargetTime, AssignmentRule, EscalationRule, EscalationRun).
Application in Features/Sla/SlaAdministration.cs (CRUD) and Features/Sla/SlaEngine.cs
(SlaCalculator, AssignmentEngine, EscalationExecutor, event handlers, EvaluateSlaCommand).
Contracts in Contracts/Sla/SlaContracts.cs.

Endpoints and permissions:
- GET    /sla-policies                       authenticated      List (default first, then name)
- POST   /sla-policies                       sla.manage         Create {name, description, isActive, isDefault,
                                                                 categoryId, departmentId, businessHoursOnly,
                                                                 workDays[0..6], workStart/workEnd "HH:mm",
                                                                 targets[{priority, firstResponseMinutes, resolutionMinutes}]}
- PUT    /sla-policies/{id}                  sla.manage         Update (same body)
- DELETE /sla-policies/{id}                  sla.manage         Delete (tickets keep due dates)
- GET/POST /automation/assignment-rules      automation.manage  List (by order) / create {name, isActive, order,
                                                                 matchCategoryId, matchDepartmentId, matchChannel,
                                                                 matchPriority, matchKeyword, setDepartmentId,
                                                                 setPriority, strategy (None|SpecificAgent|
                                                                 RoundRobin|LeastLoaded), agentId}
- PUT/DELETE /automation/assignment-rules/{id}  automation.manage
- GET/POST /automation/escalation-rules      automation.manage  List / create {name, isActive, trigger (SlaWarning|
                                                                 SlaBreached|Unassigned|NoAgentReply), target,
                                                                 afterMinutes, matchPriority, matchDepartmentId,
                                                                 escalateTicket, raisePriorityTo, reassignToAgentId,
                                                                 notifyAssignee, notifyManagers, notifyUserIds[]}
- PUT/DELETE /automation/escalation-rules/{id}  automation.manage

Rules:
- Policy selection: most specific active policy (category > department), else the default policy;
  one target per priority; resolution >= first response > 0. Only one default policy.
- Due dates applied on TicketCreated and recomputed on priority/category/department change; business
  hours walk working days in the organization time zone.
- Assignment rules run on TicketCreated in Order; first match wins; can set department/priority and pick
  an agent (specific, round-robin cursor, least loaded among eligible agents).
- EvaluateSlaCommand every minute: 80% elapsed -> TicketSlaWarning, due passed -> TicketSlaBreached
  (each once); time-based escalation rules fire once per ticket (EscalationRun).
- Escalation actions: raise priority, reassign, escalate (automatic), in-app notifications.
- Errors: 404 SLA_POLICY_NOT_FOUND, 404 RULE_NOT_FOUND, 422 INVALID_SLA_POLICY / INVALID_ASSIGNMENT_RULE /
  INVALID_ESCALATION_RULE.
```

---

## Acceptance criteria

```
- [ ] SLA is a domain capability: `SlaPolicy`, `SlaTarget`, `SlaEscalationRule`.
- [ ] Ticket tracks `FirstResponseDueAt`, `ResolutionDueAt`, `FirstRespondedAt`, `ResolvedAt`.
- [ ] SLA clock respects business hours, holidays and pauses in `PendingCustomer`.
- [ ] Background job evaluates SLA and emits `SlaApproaching` / `SlaBreached` events.
- [ ] Assignment rules are configurable and ordered; fallback to department queue.
- [ ] Escalation rules act on events (reassign, raise priority, notify).
- [ ] Notification flow: event → handler → `INotificationService` → provider adapter (no direct provider calls from business handlers).
- [ ] Notification preferences per user and channel.
- [ ] Automation actions are idempotent and logged in ticket history.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** TK-01, TK-02.
- **Depends on code areas or other stories:** `Ticket` SLA fields and `EvaluateSla`, `TicketQueries.CanHandleAsync` / `EligibleAgents`, `NotificationSender`, `RecurringRequestService`, `Organization.TimeZone`, `IAuditTrail`.

## Extra notes (optional)

- Shipped in `b81ba28` together with the agent workspace (feature 04); only `Features/Sla/*` and the related domain/infrastructure belong to this story.

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- Agent dashboard, tasks, quick replies (feature 04); outbound customer channels (feature 03).
