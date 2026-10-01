# Story 36 — API keys, webhooks and providers (Story: IN-01)

> As-built plan: written after implementation in `customer-support-crm-api` commit `678ea67` (feat: add integrations, settings, audit export and operations migration). The integration files have not changed since, so **paths and line numbers refer to `678ea67` and are identical in `2956767`** (current `develop`), except the resx lines (added in `0936711`), `docs/endpoints.md` (added in `fdd93cd`) and the rate-limiter lines (`2956767`), which refer to `2956767`. Settings and audit export from the same commit are documented elsewhere.

## Prerequisites

- Platform (`65c74a3`): `/api/v1/external` route group requiring `PolicyNames.ApiClient`, `IExternalEndpoint` discovery, recurring-request scheduler (`AddRecurringRequest`).
- Customer Management (`0fd694e`): `Customer.ExternalSystem` / `ExternalId` / `LinkExternal` and the filtered unique index on `(external_system, external_id)`.
- Ticket Management (02): `TicketFactory`, `TicketMessageWriter`, `TicketQueries`, `TicketChannel.Api`, ticket domain events.
- Communication Channels (`e921626`): `CustomerResolver`, `IMessageSender` providers, `/channels/status` and `/channels/test`.
- Phase 2: permission policies, `IAuditTrail`, `ApiResults`, `GlobalExceptionHandler`.
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

1. An admin with `integrations.manage` creates scoped API keys (secret shown once) and revokes them.
2. An external system (ERP, website) calls `/api/v1/external` with `X-Api-Key` to find/upsert customers by external id and to create, read and reply to tickets.
3. An admin registers HTTPS webhooks for ticket/customer events, gets a signing secret once, rotates it, sends a test, sees the delivery log and retries a delivery.
4. Deliveries are signed, sent in the background with backoff and protected against SSRF.
5. Email/SMS/WhatsApp providers are configured in app settings and checked/tested from the channels admin endpoints.

**Deviations from the intake (the code is authoritative):**

| Feature spec | As built in `678ea67` |
|---|---|
| `ApiClient` entity, keys/OAuth clients | `ApiKey` entity only (name, display prefix, SHA-256 hash, scopes, expiry, last used, revoked). No OAuth clients |
| Scoped keys with **rotation** | Create + revoke only; rotating = create a new key and revoke the old one. Scopes fixed to `customers:read`, `customers:write`, `tickets:read`, `tickets:write`, immutable after creation |
| Rate limited | No dedicated policy; external calls share the global limiter (partition `sub` = key id, else IP; `src/CustomerSupportCrm.Api/Configuration/SecurityExtensions.cs` lines 122–139, at `2956767`; added in `2956767`, so in `678ea67` external calls had no rate limit at all) |
| `Webhooks/ (Subscribe, Unsubscribe, ListDeliveries, Redeliver)` | Create, update, rotate-secret, delete, test, list deliveries (last 100), retry delivery |
| `IErpConnector` adapter; ERP orders/invoices on tickets | **Not built.** ERP sync is push-only: the ERP calls `PUT /external/customers/{system}/{externalId}`. No connector interface, no ERP data on tickets/customers |
| `Providers/ (Configure, Test)` | No integrations slice. Providers are configured in `appsettings.json` `Channels` section (lines 60–83 at `2956767`) / environment; status and test via `GET /channels/status`, `POST /channels/test` (`channels.manage`, feature 03) |
| `Integration (Type, Status, Settings)` entity | **Not built** |
| Circuit breaker | **Not built.** Resilience = outbox + retry with backoff + per-request timeout |
| All secrets in a secret store | API keys: hash only. Webhook secrets: Data Protection, keys on the file system (`App_Data/keys`). Provider credentials: configuration (secret store / env vars in production, not enforced) |
| Configuration audited | Key create/revoke and webhook create/update/rotate/delete audited with string actions `integrations.*`; test and retry are not audited |
| Integration health visible | Only the per-webhook delivery log and `/channels/status`; no health endpoint or dashboard |
| Vertical slices with validators, contracts in `Contracts` | Admin endpoints are inline lambdas validated by the domain (`DomainException` → 422); request/response records declared in `IntegrationSlices.cs` (lines 23–39), not in `Contracts`. Only the two external write operations are MediatR commands with FluentValidation |

**Not in scope:** settings, audit export, OAuth, inbound provider webhooks (feature 03). **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Api/Endpoints/EndpointExtensions.cs` — lines 36–40: `/api/v1/external` group `.RequireAuthorization(PolicyNames.ApiClient)`; staff group line 18.
2. `src/CustomerSupportCrm.Application/Abstractions/Http/IEndpoint.cs` — lines 26–30 `IExternalEndpoint`; discovered in `Application/DependencyInjection.cs` line 63.
3. `src/CustomerSupportCrm.Application/Abstractions/Authorization/PolicyNames.cs` — line 16 `ApiClient = "policy:api-client"`, line 19 `ForScope(scope) => $"policy:scope:{scope}"`.
4. `src/CustomerSupportCrm.Application/Abstractions/Authentication/CrmClaimTypes.cs` — line 20 `Scope = "scope"`, line 27 `ActorTypes.ApiClient = "api_client"`.
5. `src/CustomerSupportCrm.Infrastructure/Authorization/AuthorizationSetup.cs` — lines 32–43: `ApiClient` policy and one policy per `ApiScopes.All`, both with `.AddAuthenticationSchemes("ApiKey")`.
6. `src/CustomerSupportCrm.Domain/Integrations/Integrations.cs` — `ApiScopes` (7–15), `ApiKey` (21–102), `WebhookEvents` (104–114), `WebhookSubscription` (117–183), `WebhookDelivery` (186–257). `SystemSetting`/`SystemSettings` (259–315) belong to Settings.
7. `src/CustomerSupportCrm.Application/Abstractions/Integrations/IntegrationAbstractions.cs` — `ISecretProtector` (4–9), `WebhookSendResult` + `IWebhookSender` (11–20, must refuse internal destinations), `ICurrentApiClient` (23–30).
8. `src/CustomerSupportCrm.Infrastructure/Integrations/IntegrationServices.cs` — `ApiKeyAuthenticationHandler` (24–66), `HttpCurrentApiClient` (68–77), `DataProtectionSecretProtector` (79–86), `SafeWebhookSender` (93–176).
9. `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` — lines 124–132 (Data Protection, protector, `ICurrentApiClient`, typed `HttpClient<IWebhookSender, SafeWebhookSender>` with 15 s timeout and `CreateHandler`, `AddRecurringRequest<DispatchWebhooksCommand>(15 s)`); lines 146–148 add the `ApiKey` authentication scheme next to JWT.
10. `src/CustomerSupportCrm.Application/DependencyInjection.cs` — line 54 `AddScoped<WebhookPublisher>()`.
11. `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/IntegrationConfiguration.cs` — `api_keys` (7–21, unique `KeyHash`, `Scopes text[]`, `xmin`), `webhook_subscriptions` (23–36, `Events text[]`, `xmin`), `webhook_deliveries` (38–52, payload jsonb, indexes `(Delivered, Failed, NextAttemptAt)` and `(SubscriptionId, CreatedAt)`, cascade FK). DbSets: `IApplicationDbContext.Integrations.cs` lines 8–12.
12. `src/CustomerSupportCrm.Infrastructure/Persistence/Migrations/20260930104602_AddSupportOperations.cs` — creates `api_keys` (line 42), `webhook_subscriptions` (325), `webhook_deliveries` (414, cascade FK at 434–438), index `ix_api_keys_key_hash` (974) and the two delivery indexes (1263–1271).
13. `src/CustomerSupportCrm.Domain/Roles/Permissions.cs` — line 46 `IntegrationsManage = "integrations.manage"` (in `All`, line 60; not in agent defaults).
14. `src/CustomerSupportCrm.Domain/Customers/Customer.cs` — lines 84–86 `ExternalSystem` / `ExternalId`, 168–174 `LinkExternal`; `CustomerConfiguration.cs` lines 32–34 (filtered unique index).
15. `src/CustomerSupportCrm.Application/Resources/Messages.resx` lines 231–235 (`API_KEY_NOT_FOUND`, `WEBHOOK_NOT_FOUND`) and `Messages.ar.resx` lines 306–317 (adds `INVALID_API_KEY`, `INVALID_WEBHOOK`); added in `0936711`.
16. Provider side (feature 03, `e921626`): `src/CustomerSupportCrm.Infrastructure/Channels/ChannelOptions.cs` (`Channels:Email` line 6, `Channels:WhatsApp` 31, `Channels:Sms` 51), `MessageSenders.cs` (`IsConfigured` at 23, 70, 115), `src/CustomerSupportCrm.Application/Features/Channels/CustomerMessaging.cs` lines 287–330 (`ChannelStatusResponse`, `SendChannelTestCommand`, `ChannelAdministrationEndpoints`).

---

## Backend Tasks

### 1 — Domain (`Domain/Integrations/Integrations.cs`)

- `ApiKey.Create(name, scopes, expiresAt)` (67–86): name 1–150, ≥ 1 scope, all in `ApiScopes.All`, else `DomainException(INVALID_API_KEY)`; plaintext `crm_` + 24 random bytes hex; `DisplayPrefix` = first 12 chars; `Hash` = SHA-256 hex (88). `IsUsable(now)` (90), `RecordUse(now)` throttled to 1/min (92–99), `Revoke(now)` idempotent (101).
- `WebhookSubscription.Create/Update` (157–180): name 1–150, absolute **https** URL ≤ 500, ≥ 1 known event, else `DomainException(INVALID_WEBHOOK)`; `NewSecret()` = `whsec_` + 24 random bytes hex (164); `RotateSecret` (182). Secret stored only as `ProtectedSecret`.
- `WebhookDelivery` (186–257): `MaxAttempts = 8`; `Queue`, `MarkDelivered`, `MarkAttemptFailed` (next attempt `now + 2^(Attempts-1)` minutes, `Failed` at 8, error clipped to 1000), `Retry(now)` (clears `Failed`/`Delivered`, keeps `Attempts`).

### 2 — Infrastructure (`Infrastructure/Integrations/IntegrationServices.cs`)

- `ApiKeyAuthenticationHandler` (scheme `ApiKey`, header `X-Api-Key`): no header → `NoResult`; hash lookup; unusable → `Fail` (401); `RecordUse` + save; principal with `actor=api_client`, `sub=<key id>`, `name`, one `scope` claim per scope.
- `DataProtectionSecretProtector` — purpose `CustomerSupportCrm.Secrets.v1`; app name `CustomerSupportCrm`; keys persisted to `AppContext.BaseDirectory/App_Data/keys`.
- `SafeWebhookSender` — signature `sha256=` + lowercase hex HMAC-SHA256 over `"{unixSeconds}.{payload}"`; headers `X-Crm-Event`, `X-Crm-Delivery`, `X-Crm-Timestamp`, `X-Crm-Signature`; non-2xx → failed with `HTTP nnn`; `HttpRequestException` / timeout → failed. `CreateHandler()`: `AllowAutoRedirect = false`, 10 s connect timeout, `ConnectCallback` resolves DNS and connects only to the first non-internal address (`IsInternal`: loopback, any, 10/8, 127/8, 0/8, 169.254/16, 172.16/12, 192.168/16, 100.64/10, ≥ 224, IPv6 link/site/unique-local, multicast).

### 3 — Admin endpoints (`IntegrationSlices.cs` 53–170)

`IntegrationAdministrationEndpoints : IEndpoint`, `MapGroup("/integrations").WithTags("Integrations").RequireAuthorization(Permissions.IntegrationsManage)`:

| Route | Lines | Notes |
|---|---|---|
| `GET /catalog` | 59–61 | `IntegrationCatalogResponse(ApiScopes, WebhookEvents)` |
| `GET /api-keys` | 63–66 | `ApiKeyResponse` list, newest first |
| `POST /api-keys` | 68–77 | 201 `CreatedApiKeyResponse(ApiKey, Key)`; audit `integrations.api_key_created` (name, scopes, prefix — never the key) |
| `POST /api-keys/{id}/revoke` | 79–88 | `API_KEY_NOT_FOUND`; audit `integrations.api_key_revoked` |
| `GET /webhooks` | 90–93 | ordered by name, no secret |
| `POST /webhooks` | 95–106 | 201 `WebhookWithSecretResponse`; honours `isActive`; audit `integrations.webhook_created` |
| `PUT /webhooks/{id}` | 108–117 | `WEBHOOK_NOT_FOUND`; audit `integrations.webhook_updated` |
| `POST /webhooks/{id}/rotate-secret` | 119–129 | new secret returned once; audit `integrations.webhook_secret_rotated` |
| `DELETE /webhooks/{id}` | 131–140 | cascade deletes deliveries; audit `integrations.webhook_deleted` |
| `POST /webhooks/{id}/test` | 142–151 | queues `webhook.test` delivery (sent even if the subscription has no events for it) |
| `GET /webhooks/{id}/deliveries` | 153–158 | last 100 `WebhookDeliveryResponse` (no payload) |
| `POST /deliveries/{id}/retry` | 160–168 | `WEBHOOK_DELIVERY_NOT_FOUND` (inline literal) |

Error code constants: `IntegrationMapping.ApiKeyNotFound` / `WebhookNotFound` (43–44).

### 4 — Event publication and dispatch

- `WebhookPublisher(IApplicationDbContext, TimeProvider)` (175–198): for each active subscription whose `Events` contains the type, adds a `WebhookDelivery` with payload `{id, type, occurredAt, data}` (camelCase); saved by the caller's `SaveChangesAsync` (same transaction).
- `WebhookEventHandlers` (200–251) handle `DomainEventNotification<…>` for `TicketCreated` (`channel`), `TicketStatusChanged` (`from`, `to`), `TicketAssigned` (`agentId`), `TicketMessageAdded` (skips internal notes; `messageId`, `author`, `body`), `TicketFeedbackSubmitted` (`rating`), `CustomerCreated` (`customerId`, `number`, `name`, `email`). Ticket payloads add `ticketId, number, subject, status, priority, customerId, details`.
- `DispatchWebhooksCommand` / `DispatchWebhooksHandler` (253–294): up to 50 due deliveries ordered by `NextAttemptAt`; inactive or deleted subscription → `MarkAttemptFailed("Subscription inactive.")`; sends with the unprotected secret; saves after each delivery. Scheduled every 15 s (Context item 9).

### 5 — External API (`IntegrationSlices.cs` 296–497)

- Records (298–307): `ExternalCustomerUpsertRequest`, `ExternalCustomerResponse`, `ExternalCreateTicketRequest`, `ExternalTicketResponse`, `ExternalMessageRequest`.
- `UpsertExternalCustomerCommand` + validator (309–322: system 1–50 `^[A-Za-z0-9_.-]+$`, externalId 1–100, name required, type a `CustomerType`, language `en|ar`, email format) and handler (325–372): find by `(ExternalSystem, ExternalId)`; new customer → branch from `branchCode` (`BRANCH_NOT_FOUND` if unknown/inactive) or `CustomerResolver.DefaultUnitAsync`, number from the customer sequence, `LinkExternal`; existing → `UpdateProfile`; adds email/phone contacts not yet present (source = system).
- `ExternalCreateTicketCommand` + validator (374–386: subject/description lengths, priority, `customerId` or `customerEmail` required) and handler (388–415): known `customerId` or 404 `CUSTOMER_NOT_FOUND`; else `CustomerResolver.ResolveAsync(Email, …)`; inactive/unknown category ignored; `TicketFactory.CreateAsync(..., TicketChannel.Api, ..., scope: null)`.
- `ExternalApiEndpoints : IExternalEndpoint` (422–489): `GET /customers` (filter required → 400 `REQUIRED`, max 20), `PUT /customers/{system}/{externalId}`, `POST /tickets` (201, `Location: /api/v1/external/tickets/{number}`), `GET /tickets/{number}`, `POST /tickets/{number}/messages` (body 1–20 000; written as a customer message on channel `Api`). Scope policies via `PolicyNamesFor` (491–497).

### 6 — Provider configuration (from the integrations angle)

No new code in this story. `appsettings.json` `Channels:Email|WhatsApp|Sms` (all `Enabled: false`) bind to `EmailOptions` / `WhatsAppOptions` / `SmsOptions`; each `IMessageSender.IsConfigured` drives `GET /channels/status`, and `POST /channels/test` queues a test outbound message. Inbound provider webhooks (`/public/channels/{channel}/webhook`) are signature-checked adapters. See `../communication-channels/` for the as-built details.

### 7 — Docs and localization

- `docs/endpoints.md` line 12 (route group table), lines 173–180 (admin routes), 213–220 (external API and scopes).
- Messages: see Context item 15. `WEBHOOK_DELIVERY_NOT_FOUND` has no resx entry (English fallback message only).

---

## Edge Cases & Failure Modes

- **Missing `X-Api-Key`** → 401 (no result, challenge); **unknown / revoked / expired key** → 401; **valid key without the scope** → 403.
- **Staff JWT on `/external`** → 401/403 (policies authenticate only with the `ApiKey` scheme).
- **Invalid scopes, blank name, http URL, unknown event** → 422 `INVALID_API_KEY` / `INVALID_WEBHOOK` (`DomainException`).
- **Lost plaintext key or secret** — cannot be read back; create a new key / rotate the secret.
- **Webhook URL resolving to a private address** — every attempt fails with "resolves to a private or internal address" until marked `Failed`; redirects are not followed.
- **Receiver down** — retried after 1, 2, 4, 8, 16, 32, 64 minutes, failed after the 8th attempt; a manual retry keeps `Attempts`, so one more failure marks it failed again.
- **Deactivated subscription** — queued deliveries fail with "Subscription inactive."; no new ones are queued.
- **Data Protection keys lost** (new container without `App_Data/keys`) — `Unprotect` throws and dispatch stops for those subscriptions; rotate secrets.
- **External upsert race** — the filtered unique index rejects a duplicate `(system, externalId)` → 409 `CONFLICT`.
- **Unknown ticket number** → 404 `TICKET_NOT_FOUND`; any `tickets:read` key can read any ticket (no per-key data scoping).
- **Concurrent key/webhook edits** — `xmin` → 409 `CONFLICT`.
- **PII in webhooks** — `ticket.message_added` carries the message body and `customer.created` the email.

---

## Test Plan

Test projects are **out of scope** (`tests/` must not be modified). No tests are added, changed or removed by this story. There are **no existing tests** that cover integrations; `678ea67` only added `builder.UseSetting("BackgroundJobs:Enabled", "false")` to `tests/CustomerSupportCrm.Api.Tests/ApiFactory.cs` and `tests/CustomerSupportCrm.IntegrationTests/PostgresApiFactory.cs` so recurring jobs (including webhook dispatch) do not run during tests.

---

## Verification Steps

1. **Backend builds:** in `customer-support-crm-api/` run `dotnet build` — 0 warnings, 0 errors.
2. **Run:** migrate, start the API, sign in as an administrator and export `TOKEN`.
3. **Catalog:** `curl https://localhost:<port>/api/v1/integrations/catalog -H "Authorization: Bearer $TOKEN"` → 4 scopes, 6 events.
4. **API key:** `curl -X POST …/api/v1/integrations/api-keys -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"name":"ERP","scopes":["customers:write","tickets:write","tickets:read"]}'` → 201 with `key` (`crm_…`); `GET …/api-keys` shows `displayPrefix` only. `{"name":"x","scopes":["admin"]}` → 422 `INVALID_API_KEY`.
5. **External API:** `curl -X PUT …/api/v1/external/customers/erp/C-1001 -H "X-Api-Key: $KEY" -d '{"name":"Acme","email":"ops@acme.test"}'` → customer with `externalSystem: "erp"`; repeat → same id. `POST …/external/tickets -d '{"customerEmail":"ops@acme.test","subject":"Order late","description":"…"}'` → 201; `GET …/external/customers?email=ops@acme.test` → 403 (no `customers:read`).
6. **Revoke:** `POST …/integrations/api-keys/{id}/revoke` → `revokedAt` set; the key now gets 401.
7. **Webhook:** `POST …/integrations/webhooks -d '{"name":"Hook","url":"https://<receiver>","events":["ticket.created"]}'` → 201 with `secret`; `http://` URL → 422 `INVALID_WEBHOOK`. Create a ticket; within ~15 s `GET …/webhooks/{id}/deliveries` shows `delivered: true`; verify `X-Crm-Signature` = HMAC-SHA256(secret, `timestamp.body`).
8. **Test / retry / rotate:** `POST …/webhooks/{id}/test` → a `webhook.test` delivery; `POST …/deliveries/{id}/retry` → re-queued; `POST …/webhooks/{id}/rotate-secret` → new secret.
9. **Providers:** `GET …/api/v1/channels/status` → `configured` per channel; `POST …/api/v1/channels/test -d '{"channel":"Email","to":"me@example.test"}'` → 200 and an outbox row.
10. **Permissions:** an Agent token on `/api/v1/integrations/api-keys` → 403.

---

## Done Criteria

- [x] External API under `/api/v1/external` uses the standard envelope and appears in OpenAPI; documented in `docs/endpoints.md`.
- [x] Scoped API keys (hash-only storage, expiry, revocation, last-used). [ ] Key rotation and a dedicated rate limit — not built (global limiter only).
- [x] Outbound webhooks: HMAC-signed payloads, retries with exponential backoff, delivery log, test and manual retry, SSRF protection.
- [ ] `IErpConnector` and ERP context on tickets — not built; ERP sync is push-only via the external upsert endpoint, with no ERP types in Domain.
- [x] Webhook failures never break core flows (outbox in the same transaction, background dispatch). [ ] Circuit breaker — not built.
- [x] Key and webhook configuration changes audited; API keys stored as hashes, webhook secrets encrypted. [ ] Provider credentials in a secret store — configuration only.
- [ ] Integration health view — not built (delivery log and `/channels/status` only).
- [x] Provider configuration and test available through the channels admin endpoints (feature 03).
- [x] Nothing changed in `docker-compose.yml`, `deploy/` or `.github/`; in `tests/` only the two factory settings noted above.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding.**
