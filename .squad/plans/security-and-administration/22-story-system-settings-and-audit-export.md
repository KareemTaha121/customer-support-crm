# Story 22 — System settings and audit log export (Story: P3-01)

> As-built plan: written after implementation in `customer-support-crm-api` commit `678ea67` (feat: add integrations, settings, audit export and operations migration (features 10-12)). Paths and line numbers refer to `678ea67`. This plan covers only the settings and audit-export parts of that commit; integrations (API keys, webhooks) are a separate story. Follow-ups: `0936711` (en/ar messages for `UNKNOWN_SETTING`, `FEATURE_DISABLED`) and `e3e5d3d` (`GET /ai/status` honors `ai.agent_assist_enabled`).

## Prerequisites

- Story 03 completed: [03-story-current-user-and-permission-authorization.md](03-story-current-user-and-permission-authorization.md) — `ICurrentUser`, one policy per permission code.
- Story 06 completed: [06-story-audit-logging.md](06-story-audit-logging.md) — `AuditLog` (append-only), `IAuditTrail.Record`, `GET /audit-logs` (`Features/AuditLogs/List`).
- Platform Story 21 completed: [../platform/21-story-organization-context-and-notifications.md](../platform/21-story-organization-context-and-notifications.md) (`65c74a3`) — `IPublicEndpoint` / `/api/v1/public`, `RecurringRequestService<TRequest>` / `AddRecurringRequest`, `audit.export` and `settings.manage` in the permission catalog.
- Feature 02 (tickets, `57e52f8`) — `Ticket.Status`, `ResolvedAt`, `ChangeStatus`, used by the auto-close job.
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

1. A code-defined **settings catalog** (feature toggles + tunables) with defaults; only overridden values are stored.
2. Staff read all settings; administrators with `settings.manage` update several keys in one call; each change is audited.
3. Anonymous clients read the **public** toggles (`GET /public/features`) to hide disabled features.
4. A MediatR behavior blocks disabled features server-side (409 `FEATURE_DISABLED`).
5. An hourly job auto-closes tickets resolved for longer than `tickets.auto_close_resolved_days`.
6. Auditors with `audit.export` download the audit log as CSV for a time window, safe to open in spreadsheets.

**Deviations from the intake (the code is authoritative):**

| Intake / feature spec | As built in `678ea67` |
|---|---|
| `Features/Settings/` with Get / Update slices (Command, Validator, Handler, Endpoint) | One file `Features/Settings/SettingsSlices.cs`. `PUT /settings` runs its logic **inline in the endpoint lambda**: no MediatR command, no FluentValidation validator, so `ValidationBehavior` and `FeatureToggleBehavior` do not apply to it |
| Response/request contracts in `Contracts/` | `SettingResponse` and `UpdateSettingsRequest` are declared in the Application file (lines 21–23), not in `CustomerSupportCrm.Contracts` |
| `GET /settings` restricted to administrators | Any staff token can read every setting (no `RequireAuthorization(Permissions.SettingsManage)` on `ListSettings`) |
| "Organization settings, lookups, feature toggles" | Settings are feature toggles and one tunable only (6 keys). Organization settings live in `PUT /organization` (Story 21). No lookups catalog |
| Audit old and new values | `settings.updated` records only `newValues: { value }`; the previous value is not captured |
| Audit export with the same filters as search | `from`, `to`, `action` only. No user, entity type or entity id filters, no paging, no `from <= to` check (an inverted window returns a header-only file) |
| Export is itself audited | The export call does not write an audit entry |
| Export uses the standard envelope | Success is a raw `text/csv` file (`Results.File`). Errors (401/403) still use the envelope |
| Localized messages | `678ea67` added no resx keys; `UNKNOWN_SETTING` (Arabic only; English keeps the parameterized message) and `FEATURE_DISABLED` came in `0936711` |

**Not in scope:** integrations from the same commit; organization profile and branches (Story 21). **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Domain/Integrations/Integrations.cs` — `SettingKind` (259–264), `SystemSetting : Entity<string>` (267–294: `Value`, `UpdatedAt`, `UpdatedBy`, `Create`, `Set`), `SystemSettings` keys and `Catalog` (297–315).
2. `src/CustomerSupportCrm.Application/Abstractions/Persistence/IApplicationDbContext.Integrations.cs` — line 14 `DbSet<SystemSetting> SystemSettings`.
3. `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/IntegrationConfiguration.cs` — lines 54–63 (`system_settings`, PK column `key` max 100, `value` max 2000).
4. `src/CustomerSupportCrm.Infrastructure/Persistence/Migrations/20260930104602_AddSupportOperations.cs` — lines 310–322 create `system_settings`.
5. `src/CustomerSupportCrm.Application/Features/Settings/SettingsSlices.cs` — whole file (185 lines).
6. `src/CustomerSupportCrm.Application/DependencyInjection.cs` — line 32 `ValidationBehavior`, line 33 `FeatureToggleBehavior` (runs after validation), line 53 `AddScoped<SettingsReader>()`.
7. `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` — line 133 `AddRecurringRequest<AutoCloseResolvedTicketsCommand>(TimeSpan.FromHours(1))`.
8. `src/CustomerSupportCrm.Application/Features/AuditLogs/ExportAuditLogs.cs` — whole file (72 lines); compare with `Features/AuditLogs/List/*` (Story 06).
9. `src/CustomerSupportCrm.Domain/Audit/AuditLog.cs` — lines 32–52 (`OccurredAt`, `ActorUserId`, `Action`, `EntityType`, `EntityId`, `CorrelationId`, `OldValues`, `NewValues`, `IpAddress`, `UserAgent`).
10. `src/CustomerSupportCrm.Domain/Roles/Permissions.cs` — `AuditExport = "audit.export"`, `SettingsManage = "settings.manage"` (catalog from `65c74a3`).
11. `docs/endpoints.md` (current tree) — lines 61–62 (`/settings`), 69 (`/audit-logs/export.csv`), 200 (`/public/features`).

---

## Backend Tasks

### 1 — Domain and persistence

In `src/CustomerSupportCrm.Domain/Integrations/Integrations.cs`:

```csharp
public enum SettingKind { Boolean, Number, Text }

public sealed class SystemSetting : Entity<string>   // Id = setting key
{
    public string Value { get; private set; }
    public DateTimeOffset UpdatedAt { get; private set; }
    public Guid? UpdatedBy { get; private set; }
    public static SystemSetting Create(string key, string value, DateTimeOffset now, Guid? by);
    public void Set(string value, DateTimeOffset now, Guid? by);
}

public static class SystemSettings
{
    public const string PortalRegistrationEnabled = "portal.registration_enabled";
    public const string LiveChatEnabled = "chat.enabled";
    public const string WebFormEnabled = "webform.enabled";
    public const string ChatbotEnabled = "chatbot.enabled";
    public const string AiAgentAssistEnabled = "ai.agent_assist_enabled";
    public const string AutoCloseResolvedDays = "tickets.auto_close_resolved_days";
    public static IReadOnlyList<(string Key, SettingKind Kind, string Default, bool Public)> Catalog { get; }
}
```

`IApplicationDbContext.Integrations.cs` / `ApplicationDbContext.Integrations.cs` add `SystemSettings`. `SystemSettingConfiguration` maps table `system_settings` (`key` PK, `value` max 2000). Created by migration `20260930104602_AddSupportOperations`. No seed rows: absence means "use the default".

### 2 — Settings reader

`SettingsReader(IApplicationDbContext db)` (`SettingsSlices.cs` lines 32–67), scoped:

- `GetAsync(key)` loads every row once per request into a dictionary (`AsNoTracking`), falls back to `Catalog.First(c => c.Key == key).Default`.
- `IsEnabledAsync(key)` — `bool.TryParse` and `true`; `GetIntAsync(key)` — invariant integer, else 0.
- `EnsureEnabledAsync(key)` — `ConflictException(SettingErrors.FeatureDisabled)` → 409 `FEATURE_DISABLED`.
- `ListAsync(publicOnly)` — one `SettingResponse(Key, Kind, Value, DefaultValue, IsPublic)` per catalog entry, in catalog order.
- `SettingErrors` (25–29): `UnknownSetting = "UNKNOWN_SETTING"`, `FeatureDisabled = "FEATURE_DISABLED"`.

### 3 — Endpoints

`SettingsEndpoints : IEndpoint` (69–122), group `/settings`, tag `Settings`:

- `GET /settings` → `ApiResults.Ok(await settings.ListAsync(false, ct))`, name `ListSettings`. Staff policy only.
- `PUT /settings` (`UpdateSettingsRequest(IReadOnlyDictionary<string, string> Values)`), name `UpdateSettings`, `.RequireAuthorization(Permissions.SettingsManage)`. For each pair: unknown key → `ValidationException` (field = key, `ErrorCode = UNKNOWN_SETTING`); value check by kind (Boolean `bool.TryParse`; Number integer 0..3650; Text length ≤ 2000) → `INVALID_VALUE`; booleans normalized to `"true"`/`"false"`; upsert `SystemSetting` with `user.UserId.Value`; `audit.Record("settings.updated", "Setting", key, newValues: new { value = normalized })`. One `SaveChangesAsync` at the end, so the first invalid key rejects the whole request and nothing is stored. Returns the list from a fresh `SettingsReader(db)` (the scoped one may hold a stale cache).

`PublicFeatureEndpoints : IPublicEndpoint` (125–133): `GET /api/v1/public/features` → dictionary of public keys to values, name `GetPublicFeatures`.

### 4 — Feature toggle behavior

`FeatureToggleBehavior<TRequest, TResponse>` (136–160), registered as an open MediatR behavior after `ValidationBehavior`. Maps request **type names** to toggle keys:

| Request | Key |
|---|---|
| `PortalRegisterCommand` (`Features/CustomerPortal/PortalAccountSlices.cs`) | `portal.registration_enabled` |
| `StartChatCommand` (`Features/Channels/LiveChat.cs`) | `chat.enabled` |
| `SubmitWebFormCommand` (`Features/Channels/InboundChannels.cs`) | `webform.enabled` |
| `ChatbotCommand`, `SummarizeTicketCommand`, `SuggestReplyCommand`, `CategorizeTicketCommand`, `SuggestSolutionsCommand` (`Features/Ai/AiSlices.cs`) | `chatbot.enabled` (chatbot) / `ai.agent_assist_enabled` (the four agent-assist commands) |

All nine type names exist in the current tree. Matching is by string, so renaming a command silently drops its toggle. Follow-up `e3e5d3d` also makes `GET /ai/status` (`AiSlices.cs` line 465) report agent assist as disabled when the toggle is off.

### 5 — Auto-close job

`AutoCloseResolvedTicketsCommand` + handler (162–185): reads `tickets.auto_close_resolved_days`; `<= 0` → no-op; otherwise closes up to 500 tickets with `Status == Resolved && ResolvedAt <= now - days` via `ticket.ChangeStatus(TicketStatus.Closed, now)`, then saves. Registered hourly in `Infrastructure/DependencyInjection.cs` line 133; disabled with `BackgroundJobs:Enabled = false`.

### 6 — Audit log export

`ExportAuditLogsEndpoint : IEndpoint` (`Features/AuditLogs/ExportAuditLogs.cs` lines 14–72):

- `MapGet("/audit-logs/export.csv", (DateTimeOffset? from, DateTimeOffset? to, string? action, ...))`, `.RequireAuthorization(Permissions.AuditExport).WithTags("Audit").WithName("ExportAuditLogs")`.
- Window: `end = to ?? now`, `start = from ?? end - 30 days`; optional exact `action` filter; `OrderByDescending(OccurredAt).Take(100_000)` (`MaxRows`, line 16); actor email via a correlated sub-select on `Users`.
- CSV header `occurred_at,actor,action,entity_type,entity_id,ip_address,correlation_id,old_values,new_values`; `OccurredAt` in ISO 8601 `"O"`; `Escape` (58–71) quotes every non-empty value, doubles quotes and prefixes `'` when a value starts with `=`, `+`, `-` or `@`.
- Response: `Results.File(UTF-8 BOM + bytes, "text/csv; charset=utf-8", "audit-log.csv")` — the BOM makes Excel read Arabic correctly. The whole file is built in memory.

### 7 — Localization (follow-up `0936711`)

`Messages.ar.resx` line 114 `FEATURE_DISABLED`, line 150 `UNKNOWN_SETTING`; `Messages.resx` line 114 `FEATURE_DISABLED` (English `UNKNOWN_SETTING` keeps the per-key message "Unknown setting '<key>'."). `GlobalExceptionHandler.ToApiError` localizes `UNKNOWN_SETTING` validation failures by code; `INVALID_VALUE` keeps the endpoint's message.

### 8 — Docs

`docs/endpoints.md` (added in `fdd93cd`) lists `GET /settings` (—), `PUT /settings` (`settings.manage`), `GET /audit-logs/export.csv` (`audit.export`) and `GET /public/features`.

---

## Edge Cases & Failure Modes

- **Unknown key** — 400 `VALIDATION_ERROR`, `errors[0] = { code: "UNKNOWN_SETTING", field: "<key>" }` (the field is camel-cased per dot segment); nothing saved.
- **`"chat.enabled": "yes"`** or a number outside 0..3650 — 400 `INVALID_VALUE`; nothing saved.
- **`"True"`** — accepted and stored as `"true"`.
- **Empty `values` or missing body field** — no changes, 200 with the current list (`request.Values ?? new Dictionary`).
- **JSON `null` value** — fails the Boolean/Number parse → `INVALID_VALUE` (no Text keys exist, so the `value.Length` path is not reachable today).
- **Setting a value equal to the default** — a row is written and audited anyway.
- **Concurrent updates of the same key** — no concurrency token on `system_settings`; last write wins, and two first-time inserts of the same key race on the primary key → 409 `CONFLICT`.
- **Disabled toggle** — the matching command returns 409 `FEATURE_DISABLED`; `/public/features` lets the web app hide the feature first.
- **`tickets.auto_close_resolved_days = 0`** — job does nothing; more than 500 eligible tickets are closed over several hourly runs.
- **Export window with no rows / `from > to`** — header-only CSV, 200.
- **Export over 100,000 rows** — silently truncated to the newest 100,000.
- **Malicious values in audit data** (`=HYPERLINK(...)`) — prefixed with `'`, quoted.
- **Agent without `audit.export`** — 403 envelope; anonymous → 401 envelope.

---

## Test Plan

Squad plans do not add or change tests (`tests/` is out of scope). `678ea67` touched only `tests/CustomerSupportCrm.Api.Tests/ApiFactory.cs` and `tests/CustomerSupportCrm.IntegrationTests/PostgresApiFactory.cs` (one line each, test host configuration). **No existing test covers** settings, feature toggles, the auto-close job or the audit export.

---

## Verification Steps

1. **Build:** `dotnet build` in `customer-support-crm-api/` — 0 warnings, 0 errors.
2. **Run:** migrate (`AddSupportOperations` applied), start the API, sign in as the bootstrap admin, export `TOKEN`.
3. **Defaults:** `curl -k https://localhost:5001/api/v1/settings -H "Authorization: Bearer $TOKEN"` → six entries, `value == defaultValue`.
4. **Public:** `curl -k https://localhost:5001/api/v1/public/features` (no token) → the four public keys only.
5. **Update:** `curl -k -X PUT …/api/v1/settings -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"values":{"chat.enabled":"False","tickets.auto_close_resolved_days":"3"}}'` → `chat.enabled = "false"`, days `"3"`. `GET /api/v1/audit-logs?action=settings.updated` → two entries.
6. **Rejects:** `-d '{"values":{"nope":"1"}}'` → 400 `UNKNOWN_SETTING`; `-d '{"values":{"chat.enabled":"maybe"}}'` → 400 `INVALID_VALUE`; with `Accept-Language: ar` → Arabic messages (after `0936711`).
7. **Toggle enforced:** with `chat.enabled = false`, start a public live chat (`docs/endpoints.md`, Public section) → 409 `FEATURE_DISABLED`.
8. **Permission:** an Agent token on `PUT /settings` → 403; on `GET /settings` → 200 (see Deviations).
9. **Export:** `curl -k -OJ "…/api/v1/audit-logs/export.csv?from=2026-09-01T00:00:00Z&action=settings.updated" -H "Authorization: Bearer $TOKEN"` → `audit-log.csv`, `Content-Type: text/csv; charset=utf-8`, BOM, header row, quoted values. Agent token → 403.

---

## Done Criteria

- [x] Code-defined settings catalog with defaults; overrides stored in `system_settings`.
- [x] `GET /settings` (staff), `PUT /settings` (`settings.manage`), `GET /public/features` (anonymous).
- [x] Values validated by kind; unknown keys rejected with `UNKNOWN_SETTING`; the whole update is all-or-nothing.
- [x] Every changed setting audited as `settings.updated` (new value only — old value not captured).
- [x] Disabled toggles enforced server-side by `FeatureToggleBehavior` (409 `FEATURE_DISABLED`).
- [x] Hourly auto-close job driven by `tickets.auto_close_resolved_days`.
- [x] `GET /audit-logs/export.csv` (`audit.export`): time window, action filter, 100,000-row cap, formula-injection guard, UTF-8 BOM.
- [ ] `GET /settings` restricted to `settings.manage` — not built (any staff can read).
- [ ] Settings update as a MediatR command with a validator, contracts in `Contracts/` — not built (inline endpoint).
- [ ] Export filters for user / entity, and an audit entry for the export itself — not built.
- [x] Messages localized en/ar (`0936711`).
- [x] Nothing changed in `docker-compose.yml`, `deploy/` or `.github/` by this story.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding.**
