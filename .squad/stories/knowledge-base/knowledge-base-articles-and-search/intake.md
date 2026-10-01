# Story intake

- Folder: `.squad/stories/knowledge-base/knowledge-base-articles-and-search/intake.md`

---

## Feature

- **Feature name (display):** Knowledge Base
- **Feature slug (folder under `plans/`):** `knowledge-base`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `KB-01`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `knowledge-base`

---

## Title

```
Knowledge base articles, categories, search and public help center
```

---

## Description

```
Staff write and publish knowledge base entries (FAQ, article, guide, solution) in English or
Arabic; customers read published public entries anonymously. All slices live in one file,
Application/Features/KnowledgeBase/KnowledgeBaseSlices.cs; contracts in
Contracts/KnowledgeBase/KnowledgeBaseContracts.cs; domain in Domain/KnowledgeBase/KnowledgeArticle.cs.

Staff endpoints (/api/v1, staff JWT):
- GET    /kb/categories                    kb.view     List categories (with article counts)
- POST   /kb/categories                    kb.manage   Create {name, nameAr, description, parentId, sortOrder, isPublic}
- PUT    /kb/categories/{id}               kb.manage   Update (same body)
- DELETE /kb/categories/{id}               kb.manage   Delete (refused while sub-categories exist)
- GET    /kb/articles                      kb.view     List: page, pageSize (max 100), search, status, type,
                                                       language (en|ar), visibility, categoryId; PaginationMeta
- GET    /kb/articles/{id}                 kb.view     Get (body, tags, translations, counters)
- POST   /kb/articles                      kb.manage   Create draft {title, slug?, summary, body, type, language,
                                                       categoryId, tags[], visibility, translationOfId}
- PUT    /kb/articles/{id}                 kb.manage   Update (same body; translationOfId ignored)
- POST   /kb/articles/{id}/publish         kb.publish  Draft/Published -> Published
- POST   /kb/articles/{id}/unpublish       kb.publish  Published -> Draft
- POST   /kb/articles/{id}/archive         kb.manage   -> Archived
- POST   /kb/articles/{id}/restore         kb.manage   Archived -> Draft
- DELETE /kb/articles/{id}                 kb.manage   Soft delete
- GET    /kb/suggestions?ticketId&limit    tickets.view  Published articles matching the ticket subject
                                                         (any term, ranked; limit 1-20, default 5)

Public endpoints (/api/v1/public, anonymous):
- GET    /kb/categories                    Public categories, counts of published public articles
- GET    /kb/articles                      Published + Public only; page, pageSize (1-50, default 20),
                                           search, type, language, categoryId
- GET    /kb/articles/{slug}               Published + Public; increments ViewCount
- POST   /kb/articles/{id}/feedback        {helpful: bool}; rate-limited ("public" policy)

Rules:
- Lifecycle Draft -> Published -> Archived via domain methods Publish / Unpublish / Archive / Restore;
  archived articles must be restored before edit or publish; empty body cannot be published
  -> 422 INVALID_ARTICLE_STATE. Invalid title/slug/content/language -> 422 INVALID_ARTICLE.
- Visibility Public (portal, chatbot, staff) or Internal (staff only).
- One language per article row (en|ar); translations linked by TranslationOfId.
- Slug derived from title when empty, lower-case, Unicode letters kept; unique across all
  articles including soft-deleted -> 409 ARTICLE_SLUG_TAKEN.
- Unknown article -> 404 ARTICLE_NOT_FOUND; unknown category -> 404 KB_CATEGORY_NOT_FOUND;
  category with children -> 409 KB_CATEGORY_IN_USE; invalid category -> 422 INVALID_KB_CATEGORY.
- Full-text search behind IKnowledgeSearch (PostgreSQL, 'simple' config, prefix terms, GIN index);
  search boxes require all terms, ticket suggestions and AI context match any term.
- Create/update and every lifecycle action write an audit entry (kb.article_*).
- Error messages localized (en/ar) in Messages.resx / Messages.ar.resx.
```

---

## Acceptance criteria

```
- [ ] Article lifecycle: Draft -> Published -> Archived; publishing is a domain action, not a generic status setter.
- [ ] Visibility: Internal (agents only) vs Public (portal).
- [ ] Localized title/body per language; fallback language rules.
- [ ] Hierarchical categories.
- [ ] Search via PostgreSQL full-text (Arabic + English configs) behind a search abstraction.
- [ ] Article versioning / last-updated tracking.
- [ ] "Was this helpful?" feedback and view counts.
- [ ] Articles can be linked to tickets and ticket categories.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** P2-01 – P2-07 (identity, permissions, audit), Platform (12) localization.
- **Depends on code areas or other stories:** `IApplicationDbContext`, `IAuditTrail`, `ICurrentUser`, `IAccessScopeProvider` (ticket scope for suggestions), `ApiResults`, `PagedResult`, `CommonRules`, `IPublicEndpoint` route group, `RateLimitPolicies`.

## Extra notes (optional)

- Feature spec: `.squad/features/06-knowledge-base.md`.
- Feeds Customer Portal (08: public help center, deflection) and AI Features (07: `KnowledgeContext` uses `IKnowledgeSearch`).
- Frontend: `.squad/plans/frontend/15-story-knowledge-base-ui.md` (FE-08).

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.
- Implemented in commit `0027cd1`; follow-up `889dfcf` (match-any for suggestions); search moved onto `SqlRunner` in `2de665a`; migration `AddSupportOperations` added in `678ea67`.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- Article images/attachments, rich-text (HTML) bodies, per-visitor feedback records.
