# Customer Portal

Feature spec: [../../features/08-customer-portal.md](../../features/08-customer-portal.md).
Repo: `customer-support-crm-api`. Out of scope for every story: Docker, deployment, CI/CD, and files under `tests/`.

| NN | Tracker id | Plan file | Title | Depends on | Status |
|----|-----------|-----------|-------|-----------|--------|
| 30 | CP-01 | [30-story-portal-accounts-and-authentication.md](30-story-portal-accounts-and-authentication.md) | Portal accounts and authentication | Security & Administration 01–07, Customer Management (01), Channels (03) | Done |
| 31 | CP-02 | [31-story-portal-tickets-and-feedback.md](31-story-portal-tickets-and-feedback.md) | Portal tickets, history and satisfaction feedback | 30, Ticket Management (02), Knowledge Base 29 | Done |
| 40 | BUG-04 | [40-story-portal-profile-update-response.md](40-story-portal-profile-update-response.md) | PUT /portal/me returns the updated name | 30 | Done (`1ad5302`) |

Story intakes: [../../stories/customer-portal/portal-accounts-and-authentication/intake.md](../../stories/customer-portal/portal-accounts-and-authentication/intake.md), [../../stories/customer-portal/portal-tickets-and-feedback/intake.md](../../stories/customer-portal/portal-tickets-and-feedback/intake.md).

Both stories were implemented together in `customer-support-crm-api` commit `00d3f35` (feat: add customer portal (feature 08)) without plans; plans 30–31 are **as-built** plans written afterwards. The portal files (`Features/CustomerPortal/*`, `Contracts/Portal/PortalContracts.cs`, `Domain/Customers/CustomerAccount.cs`) have not changed since, so line numbers match `00d3f35` and `develop` HEAD `2956767`. Related changes in other commits:

- `65c74a3` (phase 3) — customer token, `ICurrentCustomer`, `PolicyNames.Customer` and the `/api/v1/portal` group already existed.
- `0fd694e` (feature 01) — `CustomerAccount` entity and mapping.
- `678ea67` — `customer_accounts` table in migration `AddSupportOperations`; `FeatureToggleBehavior` gates self-registration on `portal.registration_enabled`.
- `0936711` — en/ar resx messages for the portal error codes.

Frontend: [../frontend/09-story-authentication-staff-and-portal.md](../frontend/09-story-authentication-staff-and-portal.md) (FE-02, portal sign-in shell) and [../frontend/17-story-customer-portal-ui.md](../frontend/17-story-customer-portal-ui.md) (FE-10, portal tickets, feedback, help). The public help center is [../frontend/15-story-knowledge-base-ui.md](../frontend/15-story-knowledge-base-ui.md).

Known gaps and drift (spec vs as built):

- **No phone/SMS OTP sign-in.** Email + password only; the 6-digit email code is used once to verify the address.
- **No company/role visibility.** A customer sees only tickets whose `CustomerId` is their own.
- **Raw ticket statuses** (`PendingInternal`, `Escalated`, …) are returned; customer-friendly mapping is left to the client.
- **Assigned agent name** is exposed in `PortalTicketResponse.AgentName` (no ids or routing data).
- **CSAT is not one-per-ticket** — feedback is accepted while Resolved or Closed and a new submission overwrites the old one. The "survey" is the resolution email from Channels (`CustomerMessenger.QueueResolvedAsync`), not a dedicated survey entity.
- **No dedicated deflection or portal KB endpoints** — the portal uses `/api/v1/public/kb/*` (story 29).
- **Revoking portal access does not end sessions** — tokens (8 h, not refreshable) stay valid; accounts are not checked per request. → BUG-05 ([intake](../../stories/customer-portal/revoke-portal-sessions-on-access-revoke/intake.md))
- ~~**`PUT /portal/me` returns the old name** — it updates `Customer.Name`, but the profile's `name` is read from `CustomerAccount.DisplayName`, which is not changed.~~ — fixed by story 40 (`1ad5302`); [intake](../../stories/customer-portal/portal-profile-update-response/intake.md)
- **Staff rename does not reach the portal** — `PUT /customers/{id}` updates `Customer.Name` but not `CustomerAccount.DisplayName`, so the portal keeps the old name after a staff edit (found while fixing BUG-04; no intake yet).
- **Portal change-password** only checks length 12–128 and does not reject reusing the current password.
- **Branding per organization** is served by Platform (`/api/v1/public/branding`), not by these stories.
- **No tests** cover the portal.
