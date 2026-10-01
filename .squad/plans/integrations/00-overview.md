# Integrations

Feature spec: [../../features/11-integrations.md](../../features/11-integrations.md).
Repo: `customer-support-crm-api`. Out of scope for every story: Docker, deployment, CI/CD, and files under `tests/`.

| NN | Tracker id | Plan file | Title | Depends on | Status |
|----|-----------|-----------|-------|-----------|--------|
| 36 | IN-01 | [36-story-api-keys-webhooks-and-providers.md](36-story-api-keys-webhooks-and-providers.md) | API keys and external API, outbound webhooks, ERP customer sync, provider configuration | Security (10), Channels (03), Customers (01), Tickets (02) | Done |

Intake: [../../stories/integrations/api-keys-webhooks-and-providers/intake.md](../../stories/integrations/api-keys-webhooks-and-providers/intake.md).

The feature was implemented directly on `develop` without a plan in `customer-support-crm-api` commit `678ea67` (feat: add integrations, settings, audit export and operations migration (features 10-12)). Only the integrations part (`Features/Integrations/IntegrationSlices.cs`, `Domain/Integrations` API key / webhook types, `Infrastructure/Integrations`, `IntegrationConfiguration.cs`, the integration tables in migration `20260930104602_AddSupportOperations`) belongs here; settings and audit export are documented with their own features. Later commits: `0936711` (en/ar messages for `API_KEY_NOT_FOUND`, `WEBHOOK_NOT_FOUND`, `INVALID_API_KEY`, `INVALID_WEBHOOK`), `fdd93cd` (endpoint catalog), `2956767` (global rate limiter, which now also covers `/external`). Plan 36 is an **as-built** plan; its line numbers refer to `678ea67` (unchanged in `2956767`, except where the plan says otherwise).

The email/SMS/WhatsApp provider adapters were built with Communication Channels in `e921626`. Plan 36 describes them only as integration configuration; the full as-built detail is in [../communication-channels/](../communication-channels/).

Frontend counterpart: [../frontend/19-story-administration-ui.md](../frontend/19-story-administration-ui.md) (`/admin/integrations`: API keys, webhooks, deliveries) and [../frontend/12-story-channels-and-live-chat-console.md](../frontend/12-story-channels-and-live-chat-console.md) (channel status/test).

Known gaps and drift:

- **No ERP connector.** There is no `IErpConnector` and no ERP data (orders, invoices, contracts) on tickets or customers. ERP sync is push-only through `PUT /api/v1/external/customers/{system}/{externalId}`.
- **No `Integration` entity, integrations catalog or health view.** Admins see only per-webhook delivery logs and `GET /channels/status`.
- **Providers are configured in `appsettings` / environment only** (`Channels:*`), not through an integrations API, and the secrets are not forced into a secret store.
- **API keys:** no rotation endpoint (create a new key, then revoke the old one), scopes cannot be edited, no OAuth clients, no per-key rate limit (global limiter only, since `2956767`; none in `678ea67`), no per-key data scoping.
- **Webhooks:** no circuit breaker; a manual retry after 8 failures gets exactly one more attempt; payloads include message bodies and customer email (PII); test and retry are not audited.
- **Data Protection keys** live on the file system (`App_Data/keys`); losing them makes stored webhook secrets unreadable.
- **Code layout:** admin endpoints are inline lambdas with domain validation, request/response records are in `IntegrationSlices.cs` rather than `Contracts`, and the audit action names are string literals, not `AuditActions` constants.
- **`WEBHOOK_DELIVERY_NOT_FOUND`** is an inline literal with no resx entry.
- **No tests** cover integrations.
