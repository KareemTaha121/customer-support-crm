# Story intake

- Folder: `.squad/stories/integrations/api-keys-webhooks-and-providers/intake.md`

---

## Feature

- **Feature name (display):** Integrations
- **Feature slug (folder under `plans/`):** `integrations`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `IN-01`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `integrations`

---

## Title

```
API keys and external API, outbound webhooks, ERP customer sync, provider configuration
```

---

## Description

```
External systems call the CRM with API keys, and admins register signed outbound
webhooks. Everything is in customer-support-crm-api
Application/Features/Integrations/IntegrationSlices.cs; domain in
Domain/Integrations/Integrations.cs; infrastructure (API-key auth handler, secret
protector, SSRF-safe webhook sender) in Infrastructure/Integrations/IntegrationServices.cs.

Admin endpoints (/api/v1/integrations, permission integrations.manage):
- GET    /catalog                         {apiScopes[], webhookEvents[]}
- GET    /api-keys                        list (newest first; never the key or hash)
- POST   /api-keys                        {name, scopes[], expiresAt?} -> 201 {apiKey, key}
                                          (plaintext key "crm_" + 48 hex, shown once)
- POST   /api-keys/{id}/revoke            sets revokedAt (idempotent)
- GET    /webhooks                        list (by name)
- POST   /webhooks                        {name, url (https), events[], isActive} -> 201 {webhook, secret}
                                          (signing secret "whsec_…", shown once)
- PUT    /webhooks/{id}                   update name, url, events, isActive
- POST   /webhooks/{id}/rotate-secret     -> {webhook, secret}
- DELETE /webhooks/{id}                   removes subscription and its deliveries
- POST   /webhooks/{id}/test              queues a "webhook.test" delivery
- GET    /webhooks/{id}/deliveries        last 100 deliveries
- POST   /deliveries/{id}/retry           re-queues a delivery now

External API (/api/v1/external, header X-Api-Key, one policy per scope):
- GET  /customers?externalSystem=&externalId=  or ?email=      customers:read (max 20)
- PUT  /customers/{system}/{externalId}                         customers:write (ERP upsert)
- POST /tickets                                                 tickets:write (channel Api;
                                                                customerId or customerEmail)
- GET  /tickets/{number}                                        tickets:read
- POST /tickets/{number}/messages {body}                        tickets:write (customer message)

Webhook events: ticket.created, ticket.status_changed, ticket.assigned,
ticket.message_added (public messages only), ticket.feedback, customer.created.

Rules:
- Only the SHA-256 hash of an API key is stored; unknown, revoked or expired key -> 401.
  Missing scope -> 403. lastUsedAt updated at most once a minute.
- Webhook deliveries are queued in the same transaction as the change (outbox) and sent by a
  recurring job every 15 s: POST JSON with X-Crm-Event, X-Crm-Delivery, X-Crm-Timestamp and
  X-Crm-Signature: sha256=HMAC-SHA256(secret, "{timestamp}.{body}").
- Retries: backoff 2^(attempt-1) minutes, failed after 8 attempts. 15 s timeout, no redirects;
  private/loopback/link-local/CGNAT/multicast addresses refused at connect time (SSRF).
- Webhook signing secrets encrypted with ASP.NET Data Protection (keys under App_Data/keys).
- Invalid key/webhook input -> 422 INVALID_API_KEY / INVALID_WEBHOOK; unknown ids -> 404
  API_KEY_NOT_FOUND / WEBHOOK_NOT_FOUND / WEBHOOK_DELIVERY_NOT_FOUND.
- Key create/revoke and webhook create/update/rotate/delete are audited.
- Email/SMS/WhatsApp providers are configured in appsettings (Channels:Email|WhatsApp|Sms)
  and checked/tested through /channels/status and /channels/test (feature 03).
```

---

## Acceptance criteria

```
- [ ] Public API is versioned, documented (OpenAPI), uses the standard response contract.
- [ ] API clients with scoped keys/permissions, rotation and revocation; rate limited.
- [ ] Outbound webhooks: signed payloads, retries with backoff, delivery log.
- [ ] ERP integration behind an adapter interface (`IErpConnector`); no ERP types leak into Domain.
- [ ] Integration failures don't break core flows (async, resilient, circuit breaker).
- [ ] All secrets stored in secret store; configuration audited.
- [ ] Integration health visible to admins.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** Security (10), Communication Channels (03), Customer Management (01), Ticket Management (02).
- **Depends on code areas or other stories:** `IExternalEndpoint` / `PolicyNames.ApiClient` route group (platform, `65c74a3`), `Customer.LinkExternal` / `ExternalSystem` / `ExternalId` (`0fd694e`), `TicketChannel.Api`, `TicketFactory`, `TicketMessageWriter`, `CustomerResolver`, ticket and customer domain events, `IAuditTrail`, recurring requests.

## Extra notes (optional)

- Feature spec: `.squad/features/11-integrations.md`.
- Provider adapters (SMTP, WhatsApp Cloud API, Twilio SMS) and their webhooks belong to feature 03 (`e921626`); see `.squad/plans/communication-channels/`.
- Frontend: `.squad/plans/frontend/19-story-administration-ui.md` (`/admin/integrations`), `12-story-channels-and-live-chat-console.md` (channel status/test).

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- System settings and audit export (same commit `678ea67`, other features).
- OAuth clients, outbound ERP pull connectors, provider configuration stored in the database.
