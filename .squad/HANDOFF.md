# Session handoff (2026-09-30, third session)

Read this first in a new session. It replaces the old conversation.

## Standing directive from the owner

- Build all 12 features in `.squad/features/` on both the **backend and the frontend**. Work autonomously and **do not ask questions**.
- Use the **squad-kit** workflow: intake → plan (`.squad/plans/<feature>/NN-story-*.md`) → implement → update `00-overview.md` and `00-index.md`.
- **Ignore Docker and unit/e2e tests.** Verification is build-level only: `dotnet build` with 0 warnings, and `npx ng build` with 0 errors and 0 warnings.
- Commit and push each repo after each meaningful step. Use conventional commits and end each message with the `Co-Authored-By` trailer.

## Repositories

| Repo | Branch | State |
|------|--------|-------|
| `CRM` (this workspace, `.squad/`) | main | Frontend intakes (`.squad/stories/frontend/`) and **all plans 08–19** written |
| `customer-support-crm-api` | develop | Backend complete for all 12 features, pushed. `docs/endpoints.md` lists every endpoint |
| `customer-support-crm-web` | main | Stories 08–19 all committed (one `feat(...)` commit per feature) and pushed. `npx ng build`: 0 errors, 0 warnings |

## Frontend status (`customer-support-crm-web`)

Conventions, ownership rules and folder/route/i18n table: **`.squad/plans/frontend/00-overview.md`**. Follow it.

| NN | Story | Folder | State |
|----|-------|--------|-------|
| 08 | Core platform shell | `core/`, `shared/` | Done, pushed |
| 09 | Authentication (staff + portal) | `features/auth`, `features/customer-portal/auth` | Done, pushed |
| 10 | Customers UI | `features/customers` | Done, pushed |
| 11 | Tickets UI | `features/tickets` | Done, pushed |
| 12 | Channels & live chat console | `features/channels` | Done, pushed |
| 13 | Agent dashboard UI | `features/dashboard` | Done, pushed |
| 14 | SLA & automation admin | `features/sla` | Done, pushed |
| 15 | Knowledge base UI | `features/knowledge-base` | Done, pushed |
| 16 | AI assistant panel | `features/ai` | Done, pushed |
| 17 | Customer portal UI | `features/customer-portal` | Done, pushed |
| 18 | Reports UI | `features/reports` | Done, pushed |
| 19 | Administration UI | `features/administration` | Done, pushed |

Integration points verified: `/knowledge-base/articles/:id` route exists for the AI panel links; `/help/articles/:slug` exists for portal chatbot links; the quick-reply picker is used by the ticket conversation and the chat console; en/ar key sets are identical for every scope.

## Next steps

All 12 features are built on backend and frontend. Remaining work is optional polish (see known gaps).

## Known gaps (low priority)

- Customer portal: CSAT keeps a "Change rating" button (the backend accepts repeat feedback); the plan asked for read-only after rating.
- Error messages: codes with several/parameterized English messages (INVALID_TICKET, FILE_TOO_LARGE, ...) are Arabic-only in resx, so Arabic shows a generic message while English keeps the specific one. FluentValidation field messages keep the English property name in Arabic ("'Category Name' لا يجب أن يكون فارغاً").
- Admin integrations: scope/event label keys replace non-alphanumerics with `_` (`customers:read` → `admin.integrations.scopes.customers_read`).
- A live chat's first visitor message is the ticket description, so it is not part of the transcript (`GET /chat/conversations/{id}/messages`).

## Resolved in the third session

- KB article suggestions on ticket details (`GET /kb/suggestions`, match-any search).
- en/ar resx messages for every feature error code; feature-coded validation failures are localized.
- Staff chat transcript endpoint `GET /chat/conversations/{id}/messages` (`chat.handle`, no `tickets.view`).
- `GET /ai/status` honours `ai.agent_assist_enabled`, so the AI panel hides when agent assist is off.
- Public `GET /public/web-forms/categories`; the portal contact form has a category picker.
- Plural forms in `TranslationService` (`{ zero, one, two, few, many, other }` objects, `params.count`).
- `/knowledge-base/articles/:id` is the canonical editor URL (`/knowledge-base/:id` redirects).
- Ticket categories page shows the Arabic parent name; dashboard shows unread notifications as a KPI card; integration action errors use `admin.errors.*`.
- Dev: `ng serve` proxies `/api` and `/hubs` (`proxy.conf.json`) so the SameSite=Strict refresh cookie survives reloads.
