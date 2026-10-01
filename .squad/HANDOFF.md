# Session handoff (2026-10-01, fourth session)

Read this first in a new session. It replaces the old conversation.

## Standing directive from the owner

- Build all 12 features in `.squad/features/` on both the **backend and the frontend**. Work autonomously and **do not ask questions**.
- Use the **squad-kit** workflow: intake → plan (`.squad/plans/<feature>/NN-story-*.md`) → implement → update `00-overview.md` and `00-index.md`.
- **Ignore Docker and unit/e2e tests.** Verification is build-level only: `dotnet build` with 0 warnings, and `npx ng build` with 0 errors and 0 warnings.
- Commit and push each repo after each meaningful step. Use conventional commits and end each message with the `Co-Authored-By` trailer.

## Repositories

| Repo | Branch | State |
|------|--------|-------|
| `CRM` (this workspace, `.squad/`) | main | Every feature has backend plans, a frontend plan and story intakes: plans 01–39, 45 intakes (36 feature stories + 9 bug stories). Index: `.squad/plans/00-index.md`; feature → plan matrix: `.squad/features/README.md` |
| `customer-support-crm-api` | develop | Backend complete for all 12 features, pushed (HEAD `3433f13`, BUG-01 to BUG-03 fixes; migration `AddTicketSlaPolicyForeignKey` not yet applied to a database). `docs/endpoints.md` lists every endpoint |
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

## Squad workspace (fourth session)

- Backend features 01–09, 11, 12 had no plans. **As-built** plans and intakes were written for them, NN **20–36** (one folder per feature under `plans/` and `stories/`), plus **22** (system settings + audit export) in `security-and-administration`. Line numbers refer to the commit named at the top of each plan.
- Each plan has a *Deviations* table (spec vs code) and each `00-overview.md` has *Known gaps*. Treat those as the backlog of spec items not built.
- Every feature spec has a `Status` line linking its backend and frontend plans. `dotnet build` on api `2956767`: 0 warnings, 0 errors.

## Next steps

All 12 features are built on backend and frontend. Remaining work: the probable bugs below, then the spec gaps listed in each plan's deviations table (write a new intake per fix).

## Backend bug stories (NN 37–45)

Each bug below has an intake; status and links in `.squad/plans/00-index.md` ("Bug fixes"). Next: BUG-04 (plan 40).

## Probable backend bugs (found while writing plans 20–36)

- ~~`StaffHub.JoinConversation` has no permission or scope check~~ — **fixed** in api `d2563dc` (BUG-01, plan 37).
- ~~`GET /tasks?ticketId=` / `?customerId=` skips scope; unknown task links fail as 500~~ — **fixed** in api `7058b62` (BUG-02, plan 38).
- ~~`tickets.sla_policy_id` has no FK to `sla_policies`~~ — **fixed** in api `3433f13` (BUG-03, plan 39; new migration `AddTicketSlaPolicyForeignKey`).
- `PUT /portal/me` updates the customer name but returns the unchanged `CustomerAccount.DisplayName` (plan 30).
- `ORGANIZATION_UNIT_INACTIVE` is never thrown, so inactive branches/departments can still be assigned (plan 21).
- Category cycles are possible (only self-parenting blocked) for ticket and KB categories (plans 26, 29).
- Missing resx entries: `CHAT_CLOSED`, `NO_ACTIVE_BRANCH`, `OUTBOUND_MESSAGE_NOT_FOUND`, `WEBHOOK_DELIVERY_NOT_FOUND`; English resx lacks some domain codes (`TICKET_CLOSED`, `INVALID_STATUS_TRANSITION`, `CATEGORY_NOT_FOUND`).
- Revoking portal access does not end portal sessions (tokens up to 8 h) (plan 30).

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
