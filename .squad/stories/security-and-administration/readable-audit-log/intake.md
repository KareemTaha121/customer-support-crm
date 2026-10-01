# Story intake

- Folder: `.squad/stories/security-and-administration/readable-audit-log/intake.md`

---

## Feature

- **Feature name (display):** Security and Administration
- **Feature slug (folder under `plans/`):** `security-and-administration`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-18`
- **Work item type:** `Bug`
- **Status:** `To do`
- **Assignee:** ``
- **Labels:** `backend`, `frontend`, `bug`, `audit`, `i18n`

---

## Title

```
The audit log shows readable entity labels and translated actions and entity types
```

---

## Description

```
Manual QA (2026-10-01, finding M5): /admin/audit is hard to read.

- The "Entity" column shows the entity type and a raw GUID
  (01a0f596-c0bf-...) instead of something a person recognizes, such as
  C-000005, T-000004 or a category name.
- Action codes (customers.created, ticket_categories.updated) and entity types
  (Customer, TicketCategory) are shown as raw codes, also in the Arabic UI.

The audit_logs table stores only action, entity_type and entity_id
(Domain/Audit/AuditLog.cs); IAuditTrail.Record has no label parameter, and
GET /audit-logs returns no label.

Fix:
- Store an entity label with each audit entry when it is recorded (new
  nullable column audit_logs.entity_label plus a migration). The callers
  already hold the entity, so they pass its number or name. The migration
  fills the label of existing rows where it can.
- Return entityLabel from GET /audit-logs and add an entity_label column to
  the CSV export.
- Web: translate action codes and entity types through i18n keys (en + ar),
  falling back to the raw code; show the label with the id as a tooltip or
  secondary line.
```

---

## Acceptance criteria

```
- [ ] New audit entries carry an entity label (customer/ticket number, user display name, or the entity's name/title).
- [ ] Existing entries get a label from the migration where the entity or its logged values allow it.
- [ ] GET /audit-logs returns entityLabel; the CSV export has an entity_label column.
- [ ] /admin/audit shows the label (id as tooltip), translated action names and entity types in en and ar, and the raw code when no translation exists.
- [ ] The entity type filter lists every entity type the API records, translated.
- [ ] `dotnet build` passes with zero warnings; `ng build` passes with zero errors and zero warnings.
```

---

## Attachments

None. Source: `.squad/qa/2026-10-01-manual-qa-report.md`, finding M5.

---

## Dependencies

- **Blocked by / related ids:** BUG-11 (plan 47) adds four audit action codes; whichever story lands second adds their labels.
- **Depends on code areas or other stories:** audit logging (plan 06), audit export (plan 22), administration UI (frontend plan 19).

## Technical hints (optional)

- Repo roots: `customer-support-crm-api/` (branch `develop`, .NET 10) and `customer-support-crm-web/` (branch `main`, Angular).
- Adds an EF Core migration (`AddAuditEntityLabel`).

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- No e2e tests.
- Filtering or searching the audit log by label.
- Translating the keys inside old/new values JSON in the detail dialog.
