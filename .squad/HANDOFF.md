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

- The staff live chat transcript is read from the linked ticket (`GET /tickets/{id}/messages`), so it needs `tickets.view`. There is no staff chat-messages endpoint.
- `ai.agent_assist_enabled` is not a public setting, so the AI panel only hides when it is explicitly "false". Otherwise the actions show `FEATURE_DISABLED`.
- Error messages: every feature code has an Arabic resx entry. Codes with several/parameterized English messages (INVALID_TICKET, FILE_TOO_LARGE, ...) are Arabic-only in resx, so Arabic shows a generic message while English keeps the specific one. Generic field codes (REQUIRED, INVALID_LENGTH, ...) use FluentValidation's own localized messages.
- Ticket details shows KB article suggestions (`GET /kb/suggestions`, match-any search on the subject) with insert-link for public articles.
- Customer portal: CSAT allows "Change rating" (the backend accepts repeat feedback), and the contact form always sends `categoryId: null`.
- KB: the article editor answers both `/knowledge-base/{id}` and `/knowledge-base/articles/{id}`. Count texts have no plural forms.
- Tickets categories page shows the parent's English name in both languages.
- Admin integrations: scope/event label keys replace non-alphanumerics with `_`. Revoke, rotate, test and retry errors use the global snackbar.
- Dashboard: unread notifications are a text line under the KPI grid, not a KPI card.
