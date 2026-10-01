# Story intake

- Folder: `.squad/stories/security-and-administration/system-settings-and-audit-export/intake.md`

---

## Feature

- **Feature name (display):** Security & Administration — Phase 3 System Configuration
- **Feature slug (folder under `plans/`):** `security-and-administration`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `P3-01`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `phase-3`, `backend`, `settings`, `audit`

---

## Title

```
System settings (feature toggles) and audit log export
```

---

## Description

```
Administrators change system settings (feature toggles and tunables) without code changes,
anonymous clients read the public toggles, and auditors export the audit log as CSV.
customer-support-crm-api, commit 678ea67 (Features/Settings/SettingsSlices.cs,
Features/AuditLogs/ExportAuditLogs.cs). The integrations part of that commit is a separate story.

Settings catalog (code-defined; key, kind, default, public):
- portal.registration_enabled       Boolean  true  public
- chat.enabled                      Boolean  true  public
- webform.enabled                   Boolean  true  public
- chatbot.enabled                   Boolean  true  public
- ai.agent_assist_enabled           Boolean  true  staff only
- tickets.auto_close_resolved_days  Number   7     staff only (0 = never)

Endpoints and permissions:
- GET /settings                staff           All settings {key, kind, value, defaultValue, isPublic}
- PUT /settings                settings.manage {values: {"<key>": "<value>", ...}}; returns the full list
- GET /public/features         anonymous       {"<public key>": "<value>"}
- GET /audit-logs/export.csv   audit.export    from, to (default: last 30 days), action;
                                               text/csv UTF-8 with BOM, newest first, max 100,000 rows

Rules:
- Unknown key -> 400 UNKNOWN_SETTING (field = the key).
- Boolean must parse as true/false (stored "true"/"false"); Number must be an integer 0..3650;
  Text max 2000 chars -> otherwise 400 INVALID_VALUE.
- Only changed keys are stored (system_settings rows); missing rows read the catalog default.
- Each changed key is audited as settings.updated.
- Disabled toggles block their commands before the handler -> 409 FEATURE_DISABLED
  (portal registration, live chat start, web form submit, chatbot, AI agent-assist commands).
- tickets.auto_close_resolved_days drives an hourly job that closes tickets resolved longer
  than that many days.
- CSV columns: occurred_at, actor (email), action, entity_type, entity_id, ip_address,
  correlation_id, old_values, new_values; values quoted, and cells starting with = + - @ are
  prefixed with ' (spreadsheet formula injection).
```

---

## Acceptance criteria

```
- [ ] As an admin, I manage system settings and feature toggles without code changes.
- [ ] System configuration changes are audited.
- [ ] As an auditor, I export the audit log (searchable and exportable).
- [ ] Export contains no secrets beyond what the audit log already stores and is safe to open in spreadsheets.
- [ ] All new messages localized (en/ar).
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** P2-03, P2-06, PL-02
- **Depends on code areas or other stories:** `IAuditTrail`, `AuditLog` / `GET /audit-logs` (P2-06), permission policies, `IPublicEndpoint`, `RecurringRequestService` (PL-02), `Ticket.ChangeStatus` (feature 02).

## Extra notes (optional)

- Feature spec: `features/10-security-and-administration.md` (System configuration, Audit logs). Phase 2 deferred "System settings, audit export → Phase 3".
- Frontend: `plans/frontend/19-story-administration-ui.md` (settings page, audit CSV export).

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- Integrations (API keys, webhooks) from the same commit.
- Organization profile, branches, departments (PL-02).
