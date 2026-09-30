# 11 — Integrations

> **Source:** AZM Squad Customer Support CRM — Core Features §11
> **Implementation phase:** Phase 13 — Advanced Integrations (provider adapters in Phase 10)
> **Build priority:** 16

## Summary

Connect the CRM with the rest of the business: expose APIs, sync with ERP, plug in messaging providers and other external systems.

## Scope

| Sub-feature | Description |
|-------------|-------------|
| APIs | Public, versioned REST API with API keys/OAuth clients and webhooks. |
| ERP | Sync customers, orders/invoices/contracts to show context on tickets. |
| Email, SMS & WhatsApp | Provider configuration for the channels in feature 03. |
| External systems | Generic outbound webhooks and inbound integration endpoints. |

## User stories

- As an **integrator**, I call the CRM API with an API key to create tickets and customers.
- As an **admin**, I register webhooks to receive ticket events in another system.
- As an **agent**, I see the customer's ERP data (e.g. orders, invoices) on the ticket.
- As an **admin**, I configure and test email/SMS/WhatsApp providers.

## Acceptance criteria

- [ ] Public API is versioned, documented (OpenAPI), uses the standard response contract.
- [ ] API clients with scoped keys/permissions, rotation and revocation; rate limited.
- [ ] Outbound webhooks: signed payloads, retries with backoff, delivery log.
- [ ] ERP integration behind an adapter interface (`IErpConnector`); no ERP types leak into Domain.
- [ ] Integration failures don't break core flows (async, resilient, circuit breaker).
- [ ] All secrets stored in secret store; configuration audited.
- [ ] Integration health visible to admins.

## Domain model

```text
Integration (Type, Status, Settings)
ApiClient
WebhookSubscription
WebhookDelivery
```

## Backend slices

```text
Features/Integrations/
├── ApiClients/ (Create, Revoke, Rotate)
├── Webhooks/ (Subscribe, Unsubscribe, ListDeliveries, Redeliver)
├── Erp/ (GetCustomerContext, SyncCustomers)
└── Providers/ (Configure, Test)
Infrastructure/Integrations/ (ERP connector, webhook dispatcher)
```

## Frontend

- Admin: integrations catalog, API clients, webhooks, provider settings, health status.
- ERP context panel in customer/ticket details.

## Dependencies

- Security (10), Communication Channels (03), Customer Management (01), Ticket Management (02).
