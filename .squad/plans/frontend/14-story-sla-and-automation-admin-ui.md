# Story 14 — SLA and automation admin UI (Story: FE-07)

## Prerequisites

- Story 08 completed: [08-story-core-platform-shell.md](08-story-core-platform-shell.md) (`ApiService`, `ApiError`, `translationResolver`, `requirePermission`, `PermissionService`, shared states, `ConfirmService`, `applyServerErrors`).
- Story 09 completed: [09-story-authentication-staff-and-portal.md](09-story-authentication-staff-and-portal.md) — form/error style to copy (`features/auth/profile.page.ts`).
- Conventions and ownership: [00-overview.md](00-overview.md). This story edits only `customer-support-crm-web/src/app/features/sla/**` and `customer-support-crm-web/public/i18n/sla/{en,ar}.json`.

---

## Story Goal

1. `/sla` shows a tabbed admin area: **SLA policies** (`sla.manage`), **Assignment rules** and **Escalation rules** (`automation.manage`). Only tabs the user may open are shown; `/sla` redirects to the first allowed tab.
2. Each tab lists its items in a table, and supports create, edit (dialog with every field the backend accepts), quick active toggle and delete with confirmation.
3. Durations (SLA targets, escalation delay) are entered as amount + unit (minutes / hours / days) and sent as minutes.
4. Server validation errors are shown on the matching field; domain (422) errors show as a toast. Full en/ar translations, RTL-safe.

Not in scope: SLA engine behaviour, ticket SLA badges (Story 11), backend changes.

---

## Context — Read These Files First

1. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Sla/SlaAdministration.cs` — endpoints ~lines 338–405 (`/sla-policies`, `/automation/assignment-rules`, `/automation/escalation-rules`: GET list (no paging), POST, PUT `/{id}`, DELETE `/{id}`); validators `SaveSlaPolicyValidator` ~62–78 (name ≤150, description ≤1000, targets not empty, first response 1–86 400 min, resolution 1–525 600 min, `HH:mm`), `SaveAssignmentRuleValidator` ~177–188 (keyword ≤200, enum names), `SaveEscalationRuleValidator` ~260–271 (`afterMinutes` 1–43 200). Error codes `SLA_POLICY_NOT_FOUND`, `RULE_NOT_FOUND` ~21–25.
2. `customer-support-crm-api/src/CustomerSupportCrm.Contracts/Sla/SlaContracts.cs` — `SlaTargetDto`, `SlaPolicyRequest/Response`, `AssignmentRuleRequest/Response` (response adds `agentName`), `EscalationRuleRequest/Response`.
3. `customer-support-crm-api/src/CustomerSupportCrm.Domain/Sla/AutomationRules.cs` — `AssignmentStrategy` ~7–14; `Update` guard ~88–97 (`SpecificAgent` needs `agentId`); `EscalationTrigger` ~142–152; guard ~226–235 (`Unassigned`/`NoAgentReply` need `afterMinutes`; `afterMinutes` is dropped for SLA triggers ~241).
4. `customer-support-crm-api/src/CustomerSupportCrm.Domain/Sla/SlaPolicy.cs` ~78–94 — business hours need ≥1 day and end > start; one target per priority, resolution ≥ first response (422 `INVALID_SLA_POLICY`).
5. `customer-support-crm-api/src/CustomerSupportCrm.Domain/Tickets/TicketEnums.cs` ~14–46 — `TicketPriority` (Low, Medium, High, Urgent), `TicketChannel` (Agent … Api), `SlaTarget` (FirstResponse, Resolution).
6. `customer-support-crm-api/src/CustomerSupportCrm.Api/Middleware/GlobalExceptionHandler.cs` ~90–93 — field paths are camelCased segments of the command property: `policy.name`, `rule.name`, `policy.targets[0].firstResponseMinutes`. The feature must strip the `policy.`/`rule.` prefix before `applyServerErrors`.
7. Pickers: `Features/Tickets/TicketCategorySlices.cs` ~87–91 (`GET /ticket-categories?includeInactive`) + `Contracts/Tickets/TicketContracts.cs` ~117–125; `Features/Branches/BranchEndpoints.cs` ~142–145 (`GET /branches?includeInactive`, departments embedded) + `Contracts/Organization/OrganizationContracts.cs` ~24–33; `Features/Users/Administration/UserAdministration.cs` ~107 and ~168 (`GET /users/lookup?search=`, max 50 rows ~136) + `Contracts/Users/UserContracts.cs` ~22.
8. `customer-support-crm-web/src/app/shared/form-errors.ts` — `applyServerErrors` (maps `[0]` → `.0`, returns unmatched messages); `core/interceptors/error.interceptor.ts` ~33–46 `describeError`.
9. `customer-support-crm-web/src/app/core/guards/auth.guards.ts` — `requirePermission`; `core/permissions/permission.service.ts` `has()`; `core/layout/navigation.ts` ~34 (menu entry already lists both permissions); `app.routes.ts` ~33 (`SLA_ROUTES`).

---

## Frontend Tasks

### 1 — Models and API

- Create file: `src/app/features/sla/sla.models.ts` — interfaces mirroring the contracts above, enum value arrays (`PRIORITIES`, `CHANNELS`, `STRATEGIES`, `TRIGGERS`, `SLA_TARGETS`, `WEEKDAYS`), picker models (`TicketCategoryOption`, `DepartmentOption`, `UserLookup`), duration helpers (`DurationUnit`, `splitMinutes`, `toMinutes`).
- Create file: `src/app/features/sla/sla.api.ts` — `SlaApi` (`providedIn: 'root'`): `listPolicies/savePolicy(id|null)/deletePolicy`, same for assignment and escalation rules (save = POST when id is null else PUT, always `{ silent: true }`), `categories()`, `branches()`, `lookupUsers(search)`.
- Create file: `src/app/features/sla/sla-lookups.service.ts` — caches categories, departments (flattened `Branch — Department`) and known user names as signals; `ensureLoaded()`, `categoryName`, `departmentName`, `userName`, `remember(users)`.

### 2 — Shared feature helpers

- Create file: `src/app/features/sla/sla-form-utils.ts` — `applySlaServerErrors(form, error, rename?)` strips `policy.`/`rule.` and renames fields (e.g. `firstResponseMinutes` → `firstResponseAmount`), then calls `applyServerErrors`; `handleSaveError` shows unmatched/422 errors through `NotificationToastService` + `describeError`.
- Create file: `src/app/features/sla/duration.pipe.ts` — `slaDuration` pipe (`90` → `1 h 30 min`, localized).
- Create file: `src/app/features/sla/user-picker.component.ts` — `<app-sla-user-picker [label] [multiple] [(ids)] [error]>`: chip grid + debounced `mat-autocomplete` on `/users/lookup`.

### 3 — Routes and shell

- File: `src/app/features/sla/sla.routes.ts` — replace the placeholder, keep `SLA_ROUTES`. Root route: `SlaShellComponent`, `resolve: { i18n: translationResolver('sla') }`; children `''` (componentless, `slaDefaultTabGuard` redirects to `policies` / `assignment-rules` / `escalation-rules` / `/forbidden`), `policies` (`requirePermission(Permissions.slaManage)`), `assignment-rules`, `escalation-rules` (`requirePermission(Permissions.automationManage)`), `**` → `''`.
- Create file: `src/app/features/sla/sla-shell.component.ts` — page header + `mat-tab-nav-bar` (only permitted links) + `router-outlet`.

### 4 — Pages and dialogs

- `sla-policies.page.ts` + `sla-policy-dialog.component.ts`: table (name + default pill, scope, schedule, targets per priority, active toggle, actions); dialog fields name, description, active, default, category, department, business-hours-only, work days (button toggles, Sun–Sat), work start/end (`type="time"`), one row per priority (enabled checkbox, first response and resolution amount+unit). Client checks mirror the domain rules.
- `assignment-rules.page.ts` + `assignment-rule-dialog.component.ts`: table ordered by `order` (order, name, conditions, actions, active, row actions); dialog fields name, active, order (default = max+1), match category/department/channel/priority/keyword, set department/priority, strategy, agent (required for `SpecificAgent`).
- `escalation-rules.page.ts` + `escalation-rule-dialog.component.ts`: table (name, trigger with target or delay, conditions, actions, active); dialog fields name, active, trigger, target (SLA triggers, empty = either), delay amount+unit (time triggers, required), match priority/department, escalate ticket, raise priority to, reassign to agent, notify assignee/managers, notify users (multi).
- Delete uses `ConfirmService.ask({ destructive: true })`; active toggle PUTs the full item with `isActive` flipped.
- Shared table styles in `sla-pages.scss`, dialog styles in `sla-dialog.scss` (logical properties only).

### 5 — Translations

- Create files: `public/i18n/sla/en.json`, `public/i18n/sla/ar.json` — identical key sets: `title`, `tabs`, `policies`, `assignment`, `escalation`, `fields`, `priority`, `channel`, `strategy`, `trigger`, `target`, `weekdays`, `units`, `duration`, `validation`, `confirm`, `toast`.

---

## Edge Cases & Failure Modes

- User holds only one of `sla.manage` / `automation.manage` → only that tab shows; `/sla` redirects to it; direct URL to the other tab → `/forbidden`.
- `GET /users/lookup` returns at most 50 users → the picker searches server-side; names of saved ids not in the cache show the fallback `sla.fields.unknownUser` (assignment rules use `agentName` from the response).
- Server field `policy.targets[i]…` indexes the **sent** array (only enabled priorities) → the dialog keeps a sent-index → form-index map.
- Domain errors (`INVALID_SLA_POLICY`, `INVALID_ASSIGNMENT_RULE`, `INVALID_ESCALATION_RULE`, 422) have no field → toast with the localized server message.
- Setting a policy as default clears the other default server-side → the list reloads after every save.
- Deleting a policy keeps existing ticket due dates (backend comment ~127) → mentioned in the confirm text.

## Test Plan

Out of scope (build-level verification only). Manual smoke: create/edit/delete each item type; user with only `automation.manage`; Arabic RTL layout.

## Verification Steps

1. **Frontend builds:** `cd customer-support-crm-web && npx ng build` — no errors or warnings in `src/app/features/sla/**`.

## Done Criteria

- [ ] SLA policies, assignment rules and escalation rules can be listed, created, edited and deleted.
- [ ] Validation errors are shown per field. en/ar + RTL.
