# Story 54 — The audit log shows readable entity labels and translated actions and entity types (Bug: BUG-18)

> Fix plan, not yet implemented. Paths and line numbers refer to customer-support-crm-api `dca992c` (`develop`) and customer-support-crm-web `06c817a` (`main`).
> Intake: [../../stories/security-and-administration/readable-audit-log/intake.md](../../stories/security-and-administration/readable-audit-log/intake.md)

## Prerequisites

- Story 06 — [06-story-audit-logging.md](06-story-audit-logging.md): `AuditLog`, `IAuditTrail`, `GET /audit-logs`.
- Story 22 — [22-story-system-settings-and-audit-export.md](22-story-system-settings-and-audit-export.md): `GET /audit-logs/export.csv`.
- Frontend story 19 — [../frontend/19-story-administration-ui.md](../frontend/19-story-administration-ui.md): `/admin/audit`.
- Related: Story 47 — [47-story-password-reset.md](47-story-password-reset.md) adds the action codes `auth.password.reset_requested`, `auth.password.reset`, `customers.portal_password_reset_requested` and `customers.portal_password_reset`. If story 47 has landed, give its calls an `entityLabel` too, and keep their keys in the i18n table below. If it has not, story 47 adds them.
- Source: manual QA report `.squad/qa/2026-10-01-manual-qa-report.md`, finding M5.
- Backend paths are relative to **`customer-support-crm-api/`**, frontend paths to **`customer-support-crm-web/`**.

---

## Story Goal

| | Before | After |
|---|---|---|
| "Entity" column | `Customer` + `01a0f596-c0bf-…` | `Customer · C-000005` (the id in a tooltip) |
| "Action" column | `customers.created` in both languages | "Customer created" / "إنشاء عميل" (the raw code in a tooltip) |
| Entity type filter | 11 raw type names, 5 recorded types missing | All 16 recorded types, translated |
| `GET /audit-logs` | No label | `entityLabel` |
| CSV export | `entity_type,entity_id` | `entity_type,entity_id,entity_label` |

**Why it happens.** `audit_logs` stores only `action`, `entity_type` and `entity_id` (`Domain/Audit/AuditLog.cs` 32–52). `IAuditTrail.Record` has no label parameter (`Abstractions/Auditing/IAuditTrail.cs` 13–19). `AuditLogResponse` has no label (`Contracts/Audit/AuditContracts.cs` 5–17). The page prints `entityType` and `entityId` as they are (`audit.page.ts` 124–132) and the action in a monospace span (120–123). The web has no i18n keys for actions or entity types: `admin.audit` in `public/i18n/admin/en.json` 328–347 holds only column titles.

### Decision: store the label when the entry is recorded

Two options were weighed:

- **Resolve at query time.** For each page, group rows by `entity_type` and look up numbers and names in the 16 source tables. This has no migration, but:
  - It fails for **hard-deleted** entities, which are exactly the rows an auditor looks for: `roles.deleted` (`DeleteRoleHandler`), `webhook_deleted`, `sla_policies.deleted`, `*_rules.deleted` (`SlaAdministration.cs` 325–344, `db.…Remove`). Customers, tickets and KB articles are soft-deleted (`ISoftDeletable`) and would still resolve.
  - It shows today's name, not the name at the time of the action. A renamed category would rewrite its own history.
  - The CSV export reads up to 100,000 rows (`ExportAuditLogs.cs` 16), and each needs the same lookups.
- **Record time (chosen).** One nullable `entity_label` column. Every caller already has the entity loaded, so it passes its number or name. Exceptions: four call sites have only an id, and do one small lookup (listed below). The label is historical, survives deletes, and costs nothing to read. A one-time backfill in the migration labels most existing rows.

**Deviation from the intake:** none.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Domain/Audit/AuditLog.cs`: max lengths (12–17), properties (32–52), `Record` factory (54–82), `Truncate` (84–85). `AuditActions.cs` / `AuditEntityTypes` (constants for `User` and `Role` only).
2. `src/CustomerSupportCrm.Application/Abstractions/Auditing/IAuditTrail.cs` 10–20 and `src/CustomerSupportCrm.Infrastructure/Auditing/AuditTrail.cs` 20–41.
3. `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/AuditLogConfiguration.cs` 15–28.
4. `src/CustomerSupportCrm.Application/Features/AuditLogs/List/ListAuditLogsHandler.cs` 49–81 (projection and mapping), `ListAuditLogsQuery.cs`, `ListAuditLogsEndpoint.cs`.
5. `src/CustomerSupportCrm.Application/Features/AuditLogs/ExportAuditLogs.cs` 29–50.
6. `src/CustomerSupportCrm.Contracts/Audit/AuditContracts.cs` 5–17.
7. The call sites in the table under Backend task 3. There are 51 `audit.Record(` calls in 24 files: `grep -rn "audit.Record(" src/CustomerSupportCrm.Application`.
8. `src/CustomerSupportCrm.Infrastructure/Persistence/Migrations/20261001032735_AddTicketSlaPolicyForeignKey.cs` 14: an existing data fix with `migrationBuilder.Sql`.
9. Web: `src/app/features/administration/audit/audit.page.ts`: entity type filter (84–92), action column (120–123), entity column (124–132), `entityTypes` (171). `audit-detail-dialog.component.ts` 22–27. `administration.models.ts`: `AuditLogResponse` (140–153), `AUDIT_ENTITY_TYPES` (255). `public/i18n/admin/en.json` and `ar.json`, `audit` at 328.
10. `src/app/core/localization/translation.service.ts`: `translate` returns the key itself when it is missing (60–74); `has(key)` (82–84); `flatten` joins nested keys with dots (133–143), so a JSON key that itself contains dots (`"customers.created"`) is reachable as `admin.audit.actions.customers.created`.

---

## Backend Tasks

### 1 — Domain, interface, persistence

- `AuditLog`: `public const int EntityLabelMaxLength = 200;` and `public string? EntityLabel { get; private set; }`. Add the parameter `string? entityLabel` to `Record` after `entityId`, stored as `Truncate(entityLabel?.Trim(), EntityLabelMaxLength)`.
- `IAuditTrail.Record`: add **`string? entityLabel = null` as the last optional parameter**, after `actorUserId`. Several calls pass `oldValues`/`newValues` by position (for example `BranchEndpoints.cs` 94, `DepartmentEndpoints.cs` 58, `OrganizationProfile.cs` 74, `UserAdministration.cs` 40). Putting the new parameter last means none of them shift, and every call adds `entityLabel: …` by name. Document it as: "A short human-readable name of the entity at the time of the action (number, name or title). Not a secret, not free text from the request."
- `AuditTrail.Record`: pass it through to `AuditLog.Record`.
- `AuditLogConfiguration`: `builder.Property(entry => entry.EntityLabel).HasMaxLength(AuditLog.EntityLabelMaxLength);`. No index.

### 2 — Migration with backfill

Generate it **on top of the latest migration in `develop` when this story is implemented**. Story 47 and other stories may add migrations first.

```
dotnet ef migrations add AddAuditEntityLabel --project src/CustomerSupportCrm.Infrastructure --startup-project src/CustomerSupportCrm.Api --output-dir Persistence/Migrations
```

After the generated `AddColumn` (`entity_label`, `character varying(200)`, nullable), add one `migrationBuilder.Sql(…)` to fill the labels of existing rows. Each statement touches only rows where `entity_label IS NULL`:

```sql
UPDATE audit_logs a SET entity_label = c.number       FROM customers c              WHERE a.entity_label IS NULL AND a.entity_type = 'Customer'         AND a.entity_id = c.id::text;
UPDATE audit_logs a SET entity_label = t.number       FROM tickets t                WHERE a.entity_label IS NULL AND a.entity_type = 'Ticket'           AND a.entity_id = t.id::text;
UPDATE audit_logs a SET entity_label = u.display_name FROM users u                  WHERE a.entity_label IS NULL AND a.entity_type = 'User'             AND a.entity_id = u.id::text;
UPDATE audit_logs a SET entity_label = r.name         FROM roles r                  WHERE a.entity_label IS NULL AND a.entity_type = 'Role'             AND a.entity_id = r.id::text;
UPDATE audit_logs a SET entity_label = b.name         FROM branches b               WHERE a.entity_label IS NULL AND a.entity_type = 'Branch'           AND a.entity_id = b.id::text;
UPDATE audit_logs a SET entity_label = d.name         FROM departments d            WHERE a.entity_label IS NULL AND a.entity_type = 'Department'       AND a.entity_id = d.id::text;
UPDATE audit_logs a SET entity_label = k.title        FROM knowledge_articles k     WHERE a.entity_label IS NULL AND a.entity_type = 'KnowledgeArticle' AND a.entity_id = k.id::text;
UPDATE audit_logs a SET entity_label = tc.name        FROM ticket_categories tc     WHERE a.entity_label IS NULL AND a.entity_type = 'TicketCategory'   AND a.entity_id = tc.id::text;
UPDATE audit_logs a SET entity_label = s.name         FROM sla_policies s           WHERE a.entity_label IS NULL AND a.entity_type = 'SlaPolicy'        AND a.entity_id = s.id::text;
UPDATE audit_logs a SET entity_label = ar.name        FROM assignment_rules ar      WHERE a.entity_label IS NULL AND a.entity_type = 'AssignmentRule'   AND a.entity_id = ar.id::text;
UPDATE audit_logs a SET entity_label = er.name        FROM escalation_rules er      WHERE a.entity_label IS NULL AND a.entity_type = 'EscalationRule'   AND a.entity_id = er.id::text;
UPDATE audit_logs a SET entity_label = k.name         FROM api_keys k               WHERE a.entity_label IS NULL AND a.entity_type = 'ApiKey'           AND a.entity_id = k.id::text;
UPDATE audit_logs a SET entity_label = w.name         FROM webhook_subscriptions w  WHERE a.entity_label IS NULL AND a.entity_type = 'Webhook'          AND a.entity_id = w.id::text;
UPDATE audit_logs a SET entity_label = o.name         FROM organizations o          WHERE a.entity_label IS NULL AND a.entity_type = 'Organization'     AND a.entity_id = o.id::text;
-- Hard-deleted rows: fall back to the values logged with the entry (camelCase JSON, see AuditTrail.JsonOptions).
UPDATE audit_logs SET entity_label = left(COALESCE(old_values->>'number', old_values->>'name', old_values->>'title', new_values->>'number', new_values->>'name', new_values->>'title'), 200)
 WHERE entity_label IS NULL AND entity_id IS NOT NULL;
-- Failed sign-ins with an unknown email (LoginHandler.cs 38) have no entity id; label them with the email that was tried.
UPDATE audit_logs SET entity_label = left(new_values->>'email', 200)
 WHERE entity_label IS NULL AND entity_type = 'User' AND entity_id IS NULL;
```

Before committing, check every table and column name against the model snapshot. The names above come from the `ToTable(...)` calls in `Persistence/Configurations/*` and the snake-case convention (`Infrastructure/DependencyInjection.cs` 76). Labels that stay `NULL` (for example `Setting`, `AutomationRule`) are shown by their id, as today. `Down` drops the column only.

### 3 — Call sites

Add `entityLabel:` to every `audit.Record(` call. Label rules: **customers and tickets → `Number`**, **users → `DisplayName`**, **everything else → `Name`**, except KB articles (`Title`). `Setting` gets no label, because its `entity_id` is already the readable key (`SettingsSlices.cs` 112).

| File (under `src/CustomerSupportCrm.Application/Features/`) | Lines | `entityLabel:` |
|---|---|---|
| `Authentication/ChangePassword/ChangePasswordHandler.cs` | 52 | `user.DisplayName` |
| `Authentication/Login/LoginHandler.cs` | 38 | `email` (unknown email, no entity id) |
| `Authentication/Login/LoginHandler.cs` | 45, 54, 61, 73 | `user.DisplayName` |
| `Authentication/Logout/LogoutHandler.cs` | 35 | lookup¹ |
| `Authentication/Refresh/RefreshSessionHandler.cs` | 39 | lookup¹ |
| `Users/Administration/UserAdministration.cs` | 40, 67, 102 | `user.DisplayName` |
| `Users/Create/CreateUserHandler.cs` | 41 | `user.DisplayName` |
| `Users/Disable/DisableUserHandler.cs` | 48 | `user.DisplayName` |
| `Users/Enable/EnableUserHandler.cs` | 27 | `user.DisplayName` |
| `Users/SetRoles/SetUserRolesHandler.cs` | 39 | `user.DisplayName` |
| `Roles/Create/CreateRoleHandler.cs`, `Roles/Update/UpdateRoleHandler.cs`, `Roles/Delete/DeleteRoleHandler.cs` | 24, 33, 29 | `role.Name` |
| `Branches/BranchEndpoints.cs` | 94, 100, 128 | `branch.Name` |
| `Departments/DepartmentEndpoints.cs` | 58, 64, 92 | `department.Name` |
| `Customers/CustomerProfileSlices.cs` | 99, 153, 177 | `customer.Number` |
| `Customers/CustomerDetailSlices.cs` | 74, 91 | `customer.Number` |
| `Customers/CustomerDetailSlices.cs` | 286 | lookup² |
| `CustomerPortal/PortalAccountSlices.cs` | 360 | `customer.Number` |
| `CustomerPortal/PortalAccountSlices.cs` | 377 | lookup² |
| `Integrations/IntegrationSlices.cs` | 72, 83 | `key.Name` |
| `Integrations/IntegrationSlices.cs` | 101, 112, 124, 135 | `webhook.Name` |
| `KnowledgeBase/KnowledgeBaseSlices.cs` | 300, 334 | `article.Title` |
| `Organization/OrganizationProfile.cs` | 74, 100, 122 | `organization.Name` |
| `Settings/SettingsSlices.cs` | 112 | none |
| `Sla/SlaAdministration.cs` | 125, 141 | `policy.Name` |
| `Sla/SlaAdministration.cs` | 233, 317 | `rule.Name` |
| `Sla/SlaAdministration.cs` | 343 | capture³ |
| `Tickets/TicketCategorySlices.cs` | 98 | `category.Name` |
| `Tickets/TicketCommandSlices.cs` | 322 | `ticket.Number` |

¹ Only the refresh token is loaded. Add `var name = await db.Users.Where(u => u.Id == stored.UserId).Select(u => u.DisplayName).FirstOrDefaultAsync(cancellationToken);` before the call.
² Only the customer id is loaded (`EnsureAccessibleAsync`). Add `await db.Customers.Where(c => c.Id == request.CustomerId).Select(c => c.Number).FirstOrDefaultAsync(cancellationToken)`. The query filter excludes soft-deleted customers, and so does the access check just before.
³ `DeleteAutomationRuleHandler` loads the rule inside each branch. Declare `string name;` before the `if` and set `name = rule.Name;` in both branches.

For the story 47 codes (if present): `user.DisplayName` for staff, and the customer number for portal entries. The portal handlers hold the `CustomerAccount`, so look up the number the same way as in footnote ².

### 4 — Read side

- `AuditLogResponse`: add `string? EntityLabel` **after `EntityId`**. The web reads JSON by name, so the position change is safe.
- `ListAuditLogsHandler`: project `a.EntityLabel` (after `a.EntityId`, line 60) and pass `row.EntityLabel` in the mapping (75–76).
- `ExportAuditLogs`: project `a.EntityLabel`. Change the header to `occurred_at,actor,action,entity_type,entity_id,entity_label,ip_address,correlation_id,old_values,new_values` and add `Escape(r.EntityLabel)` after `Escape(r.EntityId)` (line 48). `Escape` already neutralizes leading `= + - @` (58–71), which matters because names are user input. The export keeps raw codes, not translations: it is a machine-readable file, and the codes are stable (`AuditActions.cs`: "Never rename an existing value").

**No changes to:** resx (no new error codes) or the list filters.

## Frontend Tasks

All changes stay in `src/app/features/administration/**` and `public/i18n/admin/{en,ar}.json` (the frontend overview's ownership rule).

### 1 — Model

- `administration.models.ts` `AuditLogResponse` (140–153): add `entityLabel: string | null;` after `entityId`.
- `AUDIT_ENTITY_TYPES` (255): the list has 11 of the 16 types the API records. Add `'TicketCategory'`, `'KnowledgeArticle'`, `'AssignmentRule'`, `'EscalationRule'` and `'AutomationRule'`.

### 2 — Label helper

New `features/administration/audit/audit-labels.ts`:

```ts
import { TranslationService } from '../../../core/localization/translation.service';

/** Translated audit action, or the raw code when there is no key (new or dynamic codes). */
export function auditActionLabel(translations: TranslationService, action: string): string {
  const key = `admin.audit.actions.${action}`;
  return translations.has(key) ? translations.t(key) : action;
}

export function auditEntityTypeLabel(translations: TranslationService, type: string): string {
  const key = `admin.audit.entityTypes.${type}`;
  return translations.has(key) ? translations.t(key) : type;
}
```

`translations.t` alone is not enough: it returns the full key when the key is missing (`translation.service.ts` 60–74), which would show `admin.audit.actions.x.y` instead of the code. The page and the dialog expose these as methods: `actionLabel(code)` and `typeLabel(type)`. Because `translations.has()` reads the `dictionary` signal, the OnPush templates re-render when the language changes.

### 3 — `audit.page.ts`

- **Action cell** (120–123): `<span [matTooltip]="entryOf(entry).action">{{ actionLabel(entryOf(entry).action) }}</span>`. Drop `admin-mono`/`dir="ltr"` for translated text; keep them only when the label equals the raw code.
- **Entity cell** (124–132): the first line is `typeLabel(entityType)`. Then:
  - with a label: `<div [matTooltip]="entityId" [attr.dir]="'auto'">{{ entityLabel }}</div>`;
  - without a label but with an id: the existing mono id line.
  
  `dir="auto"` keeps Arabic names and Latin numbers readable in both directions.
- **Entity type filter** (88–90): `{{ typeLabel(type) }}` as the option text, keeping `[value]="type"`.
- The free-text action filter (81–83) stays a raw-code input; it filters by exact code (`ListAuditLogsHandler.cs` 18–21).

### 4 — `audit-detail-dialog.component.ts`

- The action row (22–23) shows `actionLabel(entry.action)`, with the raw code in a muted mono line below it.
- The entity type row (24–25) shows `typeLabel(entry.entityType)`.
- Add a row before the entity id (26): `<dt>{{ 'admin.audit.entityLabel' | t }}</dt><dd dir="auto">{{ entry.entityLabel ?? '—' }}</dd>`.
- Inject `TranslationService`.

### 5 — i18n (`public/i18n/admin/en.json` and `ar.json`, under `audit`)

New keys: `admin.audit.entityLabel` ("Entity" / "الكيان"; the column header `entity` stays as is), `admin.audit.entityTypes.*` and `admin.audit.actions.*`. Write the action keys as flat dotted JSON keys, so they match the codes exactly: `"actions": { "customers.created": "Customer created", … }`.

Entity types:

| Key | en | ar |
|---|---|---|
| `User` | User | مستخدم |
| `Role` | Role | دور |
| `Organization` | Organization | المؤسسة |
| `Branch` | Branch | فرع |
| `Department` | Department | قسم |
| `Setting` | Setting | إعداد |
| `ApiKey` | API key | مفتاح API |
| `Webhook` | Webhook | Webhook |
| `Ticket` | Ticket | تذكرة |
| `TicketCategory` | Ticket category | تصنيف التذاكر |
| `Customer` | Customer | عميل |
| `KnowledgeArticle` | Knowledge article | مقال معرفي |
| `SlaPolicy` | SLA policy | سياسة SLA |
| `AssignmentRule` | Assignment rule | قاعدة توزيع |
| `EscalationRule` | Escalation rule | قاعدة تصعيد |
| `AutomationRule` | Automation rule | قاعدة أتمتة |

Actions (all codes recorded at `dca992c`; `kb.article_*` and `*_rules.deleted` are built at run time, at `KnowledgeBaseSlices.cs` 334 and `SlaAdministration.cs` 343):

| Code | en | ar |
|---|---|---|
| `auth.login.succeeded` | Signed in | تسجيل دخول |
| `auth.login.failed` | Sign-in failed | فشل تسجيل الدخول |
| `auth.login.locked_out` | Sign-in blocked (locked) | حظر تسجيل الدخول (مقفل) |
| `auth.logout` | Signed out | تسجيل خروج |
| `auth.refresh.reuse_detected` | Session token reuse detected | اكتشاف إعادة استخدام رمز الجلسة |
| `auth.password.changed` | Password changed | تغيير كلمة المرور |
| `users.created` | User created | إنشاء مستخدم |
| `users.updated` | User updated | تعديل مستخدم |
| `users.roles_changed` | User roles changed | تغيير أدوار المستخدم |
| `users.scopes_changed` | User access scope changed | تغيير نطاق وصول المستخدم |
| `users.password_reset` | Password reset by administrator | إعادة تعيين كلمة المرور بواسطة المسؤول |
| `users.disabled` | User deactivated | تعطيل مستخدم |
| `users.enabled` | User activated | تفعيل مستخدم |
| `roles.created` | Role created | إنشاء دور |
| `roles.updated` | Role updated | تعديل دور |
| `roles.deleted` | Role deleted | حذف دور |
| `branches.created` | Branch created | إنشاء فرع |
| `branches.updated` | Branch updated | تعديل فرع |
| `branches.activated` | Branch activated | تفعيل فرع |
| `branches.deactivated` | Branch deactivated | تعطيل فرع |
| `departments.created` | Department created | إنشاء قسم |
| `departments.updated` | Department updated | تعديل قسم |
| `departments.activated` | Department activated | تفعيل قسم |
| `departments.deactivated` | Department deactivated | تعطيل قسم |
| `customers.created` | Customer created | إنشاء عميل |
| `customers.updated` | Customer updated | تعديل عميل |
| `customers.deleted` | Customer deleted | حذف عميل |
| `customers.contact_saved` | Customer contact saved | حفظ جهة اتصال العميل |
| `customers.contact_removed` | Customer contact removed | حذف جهة اتصال العميل |
| `customers.attachment_deleted` | Customer attachment deleted | حذف مرفق العميل |
| `customers.portal_access_granted` | Portal access granted | منح الوصول إلى البوابة |
| `customers.portal_access_revoked` | Portal access revoked | إلغاء الوصول إلى البوابة |
| `integrations.api_key_created` | API key created | إنشاء مفتاح API |
| `integrations.api_key_revoked` | API key revoked | إلغاء مفتاح API |
| `integrations.webhook_created` | Webhook created | إنشاء Webhook |
| `integrations.webhook_updated` | Webhook updated | تعديل Webhook |
| `integrations.webhook_secret_rotated` | Webhook secret rotated | تجديد سر Webhook |
| `integrations.webhook_deleted` | Webhook deleted | حذف Webhook |
| `kb.article_created` | Article created | إنشاء مقال |
| `kb.article_updated` | Article updated | تعديل مقال |
| `kb.article_publish` | Article published | نشر مقال |
| `kb.article_unpublish` | Article unpublished | إلغاء نشر مقال |
| `kb.article_archive` | Article archived | أرشفة مقال |
| `kb.article_restore` | Article restored | استعادة مقال |
| `kb.article_delete` | Article deleted | حذف مقال |
| `organization.updated` | Organization profile updated | تعديل بيانات المؤسسة |
| `organization.branding_updated` | Branding updated | تعديل الهوية البصرية |
| `organization.logo_updated` | Logo updated | تحديث الشعار |
| `settings.updated` | Setting changed | تغيير إعداد |
| `sla_policies.created` | SLA policy created | إنشاء سياسة SLA |
| `sla_policies.updated` | SLA policy updated | تعديل سياسة SLA |
| `sla_policies.deleted` | SLA policy deleted | حذف سياسة SLA |
| `assignment_rules.created` | Assignment rule created | إنشاء قاعدة توزيع |
| `assignment_rules.updated` | Assignment rule updated | تعديل قاعدة توزيع |
| `assignment_rules.deleted` | Assignment rule deleted | حذف قاعدة توزيع |
| `escalation_rules.created` | Escalation rule created | إنشاء قاعدة تصعيد |
| `escalation_rules.updated` | Escalation rule updated | تعديل قاعدة تصعيد |
| `escalation_rules.deleted` | Escalation rule deleted | حذف قاعدة تصعيد |
| `ticket_categories.created` | Ticket category created | إنشاء تصنيف تذاكر |
| `ticket_categories.updated` | Ticket category updated | تعديل تصنيف تذاكر |
| `tickets.deleted` | Ticket deleted | حذف تذكرة |
| `auth.password.reset_requested` (story 47) | Password reset requested | طلب إعادة تعيين كلمة المرور |
| `auth.password.reset` (story 47) | Password reset by email link | إعادة تعيين كلمة المرور عبر رابط البريد |
| `customers.portal_password_reset_requested` (story 47) | Portal password reset requested | طلب إعادة تعيين كلمة مرور البوابة |
| `customers.portal_password_reset` (story 47) | Portal password reset | إعادة تعيين كلمة مرور البوابة |

Any code without a key (for example one added later) falls back to the raw code; nothing breaks.

---

## Test Plan

Test projects are **out of scope**: `tests/` must not be modified, and no e2e tests are added.

---

## Verification Steps

1. **Backend build:** `dotnet build src/CustomerSupportCrm.Api/CustomerSupportCrm.Api.csproj` gives 0 warnings and 0 errors. If a running API locks `bin/`, add `-o <temp dir>`.
2. **Migration and backfill:** `dotnet ef database update --project src/CustomerSupportCrm.Infrastructure --startup-project src/CustomerSupportCrm.Api`. Then `SELECT entity_type, count(*) FILTER (WHERE entity_label IS NULL) AS unlabeled, count(*) FROM audit_logs GROUP BY 1;` shows labels on existing `Customer`, `Ticket`, `User` and `TicketCategory` rows (the QA rows `customers.created` → `C-000005`).
3. **Frontend build:** `cd customer-support-crm-web && npx ng build` gives 0 errors and 0 warnings.
4. **New entries:** create a customer, update a ticket category, delete an SLA policy, sign in. `GET /api/v1/audit-logs?pageSize=5` shows `entityLabel` as `C-0000xx`, the category name, the policy name, and the user's display name.
5. **Deleted entity:** after `DELETE` of a role, its `roles.deleted` entry still shows the role name.
6. **UI (en):** `/admin/audit` shows "Customer created" and "Customer · C-000005". Hovering shows the raw code and the GUID. The entity type filter lists 16 translated types, and filtering by "Ticket category" works.
7. **UI (ar):** switch to العربية. Actions and types are in Arabic, the layout is RTL, and mixed Arabic/Latin labels read correctly.
8. **Fallback:** an action code with no key (for example by temporarily removing one key) shows the raw code, not `admin.audit.actions.…`.
9. **Detail dialog:** it shows the translated action, the raw code, the type, the label and the id.
10. **CSV:** Export → the header has `entity_label` after `entity_id`, and the values are filled. A label starting with `=` is exported as `'=…`.

---

## Done Criteria

- [ ] `audit_logs.entity_label` exists (migration `AddAuditEntityLabel`, generated by `dotnet ef`), with a backfill for existing rows.
- [ ] Every `audit.Record` call passes a label (except `settings.updated`), following the table above.
- [ ] `GET /audit-logs` returns `entityLabel`; the CSV has an `entity_label` column.
- [ ] The audit page and detail dialog show the label (id as tooltip), translated actions and types in en and ar, and raw codes as the fallback.
- [ ] The entity type filter lists all 16 types.
- [ ] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [ ] `dotnet build` and `ng build` pass with zero warnings.

**STOP HERE. Report to the user.**
