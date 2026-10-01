# Story 48 — Hide the AI chatbot and warn admins when no AI provider is configured (Bug: BUG-12)

> Fix plan, not yet implemented. Paths and line numbers refer to customer-support-crm-api `dca992c` (`develop`) and customer-support-crm-web `06c817a` (`main`).
> Intake: [../../stories/ai-features/hide-ai-when-unconfigured/intake.md](../../stories/ai-features/hide-ai-when-unconfigured/intake.md)

## Prerequisites

- Story 35 — [35-story-ai-assistant-and-chatbot.md](35-story-ai-assistant-and-chatbot.md): `AiAssistant`, `ChatbotHandler`, `GET /ai/status`.
- Story 22 — [../security-and-administration/22-story-system-settings-and-audit-export.md](../security-and-administration/22-story-system-settings-and-audit-export.md): `SettingsReader`, `GET /public/features`, `FeatureToggleBehavior`.
- Frontend stories 16, 17 and 19 — [../frontend/16-story-ai-assistant-panels.md](../frontend/16-story-ai-assistant-panels.md), [../frontend/17-story-customer-portal-ui.md](../frontend/17-story-customer-portal-ui.md), [../frontend/19-story-administration-ui.md](../frontend/19-story-administration-ui.md).
- Source: manual QA report `.squad/qa/2026-10-01-manual-qa-report.md`, finding H2.
- Backend paths are relative to **`customer-support-crm-api/`**, frontend paths to **`customer-support-crm-web/`**.

---

## Story Goal

`AnthropicAiClient.IsConfigured` is true only when `Ai:Enabled` is true **and** a key exists (`Ai:ApiKey` or `ANTHROPIC_API_KEY`). `appsettings.json` ships `Ai:Enabled: false`, so a fresh install has no provider. The `chatbot.enabled` setting defaults to `"true"` and is public.

| Surface | Before | After |
|---|---|---|
| `GET /public/features` → `chatbot.enabled` | The stored setting only (`"true"`) | Setting **and** provider configured |
| Portal nav "Virtual assistant", `/portal/assistant`, "other channels" list | Shown; every question answers "AI features are not configured." | Hidden; the page shows the existing "not available" view |
| `POST /public/chatbot/messages` without provider | 503 `AI_NOT_CONFIGURED`, after a knowledge base search | Same 503, checked first |
| Portal on a 503 `AI_NOT_CONFIGURED` / 409 `FEATURE_DISABLED` | Server text printed as an assistant turn | Switches to the "not available" view |
| `/admin/settings` Chatbot and AI agent assist rows | Toggles on, no hint | Warning banner plus a "No AI provider" pill on both rows |
| After saving settings | `branding.features` rebuilt from raw setting values (chatbot `"true"` again) | Public features reloaded from the server |

The staff ticket AI panel needs no change: `GET /ai/status` already returns `ai.IsConfigured && agent-assist toggle`, and the panel hides itself on `false`.

**Deviation from the intake:** none. The "provider configured" signal for the Settings page is a new endpoint (`GET /settings/status`), not a new field on `SettingResponse`, so the settings list keeps its shape.

---

## Context — Read These Files First

Backend (`customer-support-crm-api/`):

1. `src/CustomerSupportCrm.Infrastructure/Ai/AnthropicAiClient.cs` lines 33–44: the client is built only when `_options.Enabled && hasKey`; `IsConfigured => _client is not null` (44).
2. `src/CustomerSupportCrm.Application/Abstractions/Ai/IAiCompletionClient.cs` line 32: `bool IsConfigured { get; }`. Registered as a singleton in `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` line 136.
3. `src/CustomerSupportCrm.Application/Features/Settings/SettingsSlices.cs`:
   - `SettingsReader.ListAsync` (57–66);
   - `GET /settings` (75–77), staff group only, no extra permission; `PUT /settings` requires `settings.manage` (118);
   - `PublicFeatureEndpoints` (124–133): returns `ListAsync(publicOnly: true)` as a key → value dictionary, nothing else;
   - `FeatureToggleBehavior` (136–160): `ChatbotCommand` → `chatbot.enabled` gives 409 `FEATURE_DISABLED` when the toggle is off.
4. `src/CustomerSupportCrm.Domain/Integrations/Integrations.cs` lines 302–312: `ChatbotEnabled = "chatbot.enabled"` (default `"true"`, public) and `AiAgentAssistEnabled = "ai.agent_assist_enabled"` (default `"true"`, **not** public).
5. `src/CustomerSupportCrm.Application/Features/Ai/AiSlices.cs`:
   - `AiErrors.NotConfigured = "AI_NOT_CONFIGURED"` (32);
   - `AiAssistant.IsConfigured` (49) and the 503 in `RunAsync` (61–64);
   - `ChatbotHandler` (406–452): runs `KnowledgeContext.LoadAsync` (413) before `ai.RunAsync` (419);
   - `GET /ai/status` (463–467): already `ai.IsConfigured && AiAgentAssistEnabled`;
   - `ChatbotEndpoints` (504–513): `POST /public/chatbot/messages`.
6. `src/CustomerSupportCrm.Api/Middleware/GlobalExceptionHandler.cs` lines 30–31 and 57–58 (app exceptions → localized message via resx), 65 (`ServiceUnavailableException` → 503).
7. `src/CustomerSupportCrm.Application/Resources/Messages.resx` line 231 and `Messages.ar.resx` line 294: `AI_NOT_CONFIGURED` ("AI features are not configured." / Arabic). That is the text customers saw.
8. `src/CustomerSupportCrm.Api/appsettings.json` lines 87–92: `Ai:Enabled: false`, `Ai:ApiKey: ""`.

Frontend (`customer-support-crm-web/`):

9. `src/app/core/branding/branding.service.ts`: `load()` (41–52) fetches `/public/branding` and `/public/features`; `isEnabled(key)` (78–80).
10. `src/app/core/branding/feature-flags.ts` line 6: `chatbot: 'chatbot.enabled'`.
11. `src/app/features/customer-portal/portal-navigation.ts` line 19: the assistant nav item has `flag: FeatureFlags.chatbot`; `portal-shell.component.ts` line 30 hides items whose flag is off.
12. `src/app/features/customer-portal/channels/portal-unavailable.component.ts` line 62: the "other channels" list also reads the flag.
13. `src/app/features/customer-portal/customer-portal.routes.ts` line 34: `/portal/assistant` has no guard; the page itself switches on the flag.
14. `src/app/features/customer-portal/channels/portal-chatbot.page.ts`: `enabled` (62); the `error` callback (135–139) turns `describeError(...)` into a failed assistant turn with `handoff: true`. `portal-chatbot.page.html` lines 1–2 render `<app-portal-unavailable>` when `!enabled()`; the handoff buttons (50–77) offer live chat, ticket, web form or sign-in.
15. `src/app/core/interceptors/error.interceptor.ts` `describeError` (33–46): returns the first non-field server message, here the localized `AI_NOT_CONFIGURED` text.
16. `src/app/features/administration/settings/settings.page.ts`: the save-error banner (40–42), the setting rows (44–87), `load()` (167–180), and `save()` line 206, which rebuilds `branding.features` from raw public setting values.
17. `src/app/features/administration/admin.scss` lines 45–59: `.admin-banner` and `.admin-banner--info`.
18. `src/app/features/administration/administration.api.ts` lines 164–170: `listSettings`, `updateSettings`.
19. `src/app/features/ai/ticket-ai-panel.component.ts` lines 103–108: the staff panel's `visible` uses `GET /ai/status`; no change.
20. i18n: `public/i18n/admin/{en,ar}.json` → `admin.settings.*`; `public/i18n/portal/{en,ar}.json` → `portal.chatbot.unavailable` already exists.

---

## Backend Tasks

### 1 — Public features reflect the provider

In `PublicFeatureEndpoints` (`SettingsSlices.cs` 124–133), inject `IAiCompletionClient` (add `using CustomerSupportCrm.Application.Abstractions.Ai;`):

```csharp
app.MapGet("/features", async (SettingsReader settings, IAiCompletionClient ai, CancellationToken ct) =>
    {
        var features = (await settings.ListAsync(publicOnly: true, ct)).ToDictionary(s => s.Key, s => s.Value);
        // The chatbot needs a provider; without one the portal must not offer it.
        if (!ai.IsConfigured)
        {
            features[SystemSettings.ChatbotEnabled] = "false";
        }

        return ApiResults.Ok(features);
    })
```

Use `IAiCompletionClient` directly, not `AiAssistant`: `AiAssistant` is scoped and needs `ICurrentUser`, which this anonymous endpoint does not need.

### 2 — Settings status for the admin page

In `SettingsEndpoints` (`SettingsSlices.cs` 69–122), add:

```csharp
public sealed record SettingsStatusResponse(bool AiProviderConfigured);

group.MapGet("/status", (IAiCompletionClient ai) => ApiResults.Ok(new SettingsStatusResponse(ai.IsConfigured)))
    .RequireAuthorization(Permissions.SettingsManage)
    .WithName("GetSettingsStatus")
    .Produces<ApiResponse<SettingsStatusResponse>>();
```

`GET /api/v1/settings/status` requires `settings.manage`, the same permission as the `/admin/settings` route. It returns only a boolean: no key, model or other configuration.

### 3 — Chatbot: fail fast

At the top of `ChatbotHandler.Handle` (`AiSlices.cs` 410), before `KnowledgeContext.LoadAsync`:

```csharp
if (!ai.IsConfigured)
{
    throw new ServiceUnavailableException(AiErrors.NotConfigured, "AI features are not configured.");
}
```

The status (503) and code (`AI_NOT_CONFIGURED`) stay the same; the handler no longer runs a knowledge base search it cannot use. `FeatureToggleBehavior` still runs first, so a disabled toggle stays a 409 `FEATURE_DISABLED`.

**No changes to:** domain, settings catalog, stored values, `GET /ai/status`, migrations, resx (`AI_NOT_CONFIGURED` already has en/ar), or docs.

---

## Frontend Tasks

### 4 — Portal assistant: treat "not configured" as unavailable

In `portal-chatbot.page.ts`:

- Add `private readonly serverUnavailable = signal(false);` and change `enabled` (62) to
  `computed(() => this.branding.isEnabled(FeatureFlags.chatbot) && !this.serverUnavailable())`.
- In the `send()` error callback (135–139):

```ts
error: (error: unknown) => {
  this.thinking.set(false);
  const apiError = ApiError.from(error);
  // Cached features were stale (provider removed or toggle turned off since load): show the unavailable view.
  if (apiError.hasCode('AI_NOT_CONFIGURED') || apiError.hasCode('FEATURE_DISABLED')) {
    this.serverUnavailable.set(true);
    return;
  }
  const message = describeError(apiError, this.translations);
  this.entries.update((list) => [...list, { ...this.entry('assistant', message), failed: true, handoff: true }]);
},
```

The template already renders `<app-portal-unavailable [title]="'portal.chatbot.unavailable' | t" ...>` when `!enabled()`. That view lists the live chat, contact form, ticket and help center alternatives, so the "Open a ticket" / "Chat with an agent" paths stay available. No new portal i18n keys.

The nav item (`portal-navigation.ts` 19) and the alternatives list (`portal-unavailable.component.ts` 62) need no change: they follow `chatbot.enabled`, which task 1 now reports correctly. No route guard is added: the page handles a direct visit itself.

### 5 — Settings page: warn when no provider exists

`administration.models.ts`: add

```ts
/** SettingsSlices.cs SettingsStatusResponse */
export interface SettingsStatus {
  aiProviderConfigured: boolean;
}
```

`administration.api.ts`: add `settingsStatus(): Observable<SettingsStatus>` → `this.api.get<SettingsStatus>('/settings/status', SILENT)`.

`settings.page.ts`:

- `const AI_SETTING_KEYS = new Set(['chatbot.enabled', 'ai.agent_assist_enabled']);`
- `readonly aiProviderConfigured = signal<boolean | null>(null);` In `load()`, also call `settingsStatus()`. Set the signal on success; on error leave it `null` (no warning, no error state: the status is advisory).
- Above the `.settings` card, when `aiProviderConfigured() === false`:

```html
<div class="admin-banner admin-banner--info" role="status">
  <mat-icon>warning_amber</mat-icon><span>{{ 'admin.settings.aiMissing' | t }}</span>
</div>
```

- In `.setting__label`, for keys in `AI_SETTING_KEYS` when `aiProviderConfigured() === false`:
  `<span class="crm-pill crm-pill--warning" [matTooltip]="'admin.settings.aiMissing' | t">{{ 'admin.settings.aiUnavailable' | t }}</span>`.
- The toggles stay editable: an admin may turn a feature on before setting up the provider.
- `save()` line 206: replace the `branding.features.set(...)` copy with `void this.branding.load();`. That re-reads `/public/features` (and branding), so the cached flags carry the provider rule from task 1. `BrandingService` itself is unchanged.

i18n (`public/i18n/admin/en.json` / `ar.json`, under `admin.settings`):

| Key | en | ar |
|---|---|---|
| `admin.settings.aiMissing` | No AI provider is configured on the server. The chatbot and AI agent assist stay off for everyone until one is set up, even when their toggles are on. | لا يوجد مزوّد ذكاء اصطناعي مُعدّ على الخادم. يبقى المساعد الافتراضي ومساعد الذكاء الاصطناعي للموظفين متوقفَين للجميع حتى يتم إعداد مزوّد، حتى لو كانت مفاتيحهما مفعّلة. |
| `admin.settings.aiUnavailable` | No AI provider | لا يوجد مزوّد ذكاء اصطناعي |

---

## Test Plan

Test projects are **out of scope**: `tests/` must not be modified, and no e2e tests are added.

---

## Verification Steps

1. **Backend build:** `dotnet build src/CustomerSupportCrm.Api/CustomerSupportCrm.Api.csproj` gives 0 warnings and 0 errors. If a running API locks `bin/`, build with `-o <temp dir>`.
2. **Frontend build:** `cd customer-support-crm-web && npx ng build` gives 0 errors and 0 warnings.
3. **No provider** (`Ai:Enabled: false` or no key): `GET /api/v1/public/features` → `"chatbot.enabled": "false"` while `GET /api/v1/settings` shows `chatbot.enabled` = `"true"`.
4. **Portal:** reload `/portal`. There is no "Virtual assistant" nav item. Opening `/portal/assistant` directly shows "The virtual assistant is not available right now" with the alternatives, in en and ar.
5. **Stale flag:** with the portal open, remove the provider and restart the API without reloading the portal. Ask a question → the page switches to the "not available" view, and no "AI features are not configured." turn is shown.
6. **Chatbot API:** `POST /api/v1/public/chatbot/messages` → 503 `AI_NOT_CONFIGURED`. With `chatbot.enabled` off → 409 `FEATURE_DISABLED`.
7. **Admin:** `/admin/settings` shows the warning banner and the "No AI provider" pill on Chatbot and AI agent assist, in en and ar. `GET /api/v1/settings/status` → `{ "aiProviderConfigured": false }`; without `settings.manage` → 403.
8. **Save:** toggle another setting and save. The portal (same browser) still has no assistant nav item.
9. **With a provider** (`Ai:Enabled: true` and a key): `public/features` → `"true"`; the banner and pills are gone; the assistant answers.

---

## Done Criteria

- [ ] Without an AI provider, `GET /public/features` returns `chatbot.enabled` `"false"`; with a provider it follows the setting.
- [ ] The portal hides the assistant nav item and shows the "not available" view without a provider.
- [ ] A late `AI_NOT_CONFIGURED` / `FEATURE_DISABLED` switches the portal page to the "not available" view instead of showing server text.
- [ ] `GET /settings/status` (`settings.manage`) reports `aiProviderConfigured`; `/admin/settings` warns on both AI rows in en and ar.
- [ ] Saving settings reloads the public features instead of copying raw values.
- [ ] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [ ] `dotnet build` passes with zero warnings; `ng build` passes with zero errors and zero warnings.

**STOP HERE. Report to the user.**
