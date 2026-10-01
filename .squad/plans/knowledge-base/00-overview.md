# Knowledge Base

Feature spec: [../../features/06-knowledge-base.md](../../features/06-knowledge-base.md).
Repo: `customer-support-crm-api`. Out of scope for every story: Docker, deployment, CI/CD, and files under `tests/`.

| NN | Tracker id | Plan file | Title | Depends on | Status |
|----|-----------|-----------|-------|-----------|--------|
| 29 | KB-01 | [29-story-knowledge-base-articles-and-search.md](29-story-knowledge-base-articles-and-search.md) | Knowledge base articles, categories, search and public help center | Security & Administration 01–07, Ticket Management (02) | Done |
| 44 | BUG-08 | [44-story-knowledge-category-cycle-guard.md](44-story-knowledge-category-cycle-guard.md) | Validate knowledge base category parents | 29, 43 | Done (api `e6bf4d7`, web `06c817a`) |
| 60 | BUG-24 | [60-story-article-feedback-limit.md](60-story-article-feedback-limit.md) | Limit article votes per visitor and article; no votes on internal articles (QA round 2, N4) | 29 | Done (api `93d98b1`, web `79460bd`) |

Story intake: [../../stories/knowledge-base/knowledge-base-articles-and-search/intake.md](../../stories/knowledge-base/knowledge-base-articles-and-search/intake.md).

The feature was implemented in `customer-support-crm-api` commit `0027cd1` (feat: add knowledge base (feature 06)) without a plan; plan 29 is an **as-built** plan written afterwards. Follow-ups that touch it:

- `2de665a` — `PostgresKnowledgeSearch` moved from EF `SqlQuery` onto the shared `SqlRunner` (Npgsql parameters).
- `678ea67` — migration `AddSupportOperations` creates `knowledge_articles`, `knowledge_categories` and the GIN index `ix_knowledge_articles_search`.
- `889dfcf` — ticket suggestions and AI knowledge context match **any** term (`KnowledgeSearchRequest.MatchAny`); search boxes keep all-terms matching.
- `0936711` — en/ar resx messages for the KB error codes.

Plan line numbers refer to `develop` HEAD `2956767`.

Frontend: [../frontend/15-story-knowledge-base-ui.md](../frontend/15-story-knowledge-base-ui.md) (FE-08) builds the staff KB pages and the public help center on these endpoints; ticket-side suggestions are used by [../frontend/11-story-tickets-ui.md](../frontend/11-story-tickets-ui.md) / [../frontend/16-story-ai-assistant-panels.md](../frontend/16-story-ai-assistant-panels.md).

Known gaps and drift (spec vs as built):

- **No fallback language rules.** One article row per language, linked by `TranslationOfId`; responses only list sibling translations.
- **Search uses the `'simple'` configuration**, not separate Arabic/English configs (no stemming; prefix match per term). Ranked search returns at most 100 ids, so a search list's `totalCount` is capped at 100.
- **No article versioning** — only created/updated/published timestamps and users.
- **No `ArticleFeedback` entity** — anonymous helpful/not-helpful counters, not de-duplicated; feedback is accepted for any *published* article id, including internal ones.
- **No article ↔ ticket or article ↔ ticket-category links.** Suggestions search the ticket subject only (not description).
- **No images or attachments** in articles (Markdown body only).
- **Slices are in one file** (`KnowledgeBaseSlices.cs`), not the folder layout in the spec; FAQs are `Type = Faq` articles.
- ~~**Category cycles** — only direct self-parenting is rejected; the parent id is not checked for existence~~ — fixed by story 44 (api `e6bf4d7`, web `06c817a`). Before the fix an unknown parent failed at the FK as a **500** (only unique violations map to 409). [intake](../../stories/knowledge-base/knowledge-category-cycle-guard/intake.md)
- **Docs:** `docs/endpoints.md` does not list `POST /kb/articles/{id}/restore`.
- **No tests** cover the knowledge base.
