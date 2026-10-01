# 06 — Knowledge Base

> **Source:** AZM Squad Customer Support CRM — Core Features §6
> **Implementation phase:** Phase 8 — Knowledge Base
> **Status:** Done (backend + frontend) — backend plans [29](../plans/knowledge-base/00-overview.md), frontend plan [15](../plans/frontend/15-story-knowledge-base-ui.md)
> **Build priority:** 11

## Summary

A bilingual library of answers used by agents (internal) and customers (public) to solve problems faster and deflect repetitive tickets.

## Scope

| Sub-feature | Description |
|-------------|-------------|
| FAQs | Short question/answer entries grouped by category. |
| Help articles | Rich-text articles with images and attachments. |
| Solutions and guides | Step-by-step troubleshooting guides linked to ticket categories. |
| Search | Full-text search in Arabic and English. |

## User stories

- As a **KB editor**, I write an article in Arabic and English and save it as draft.
- As a **KB editor**, I publish, unpublish or archive articles.
- As an **agent**, I search the KB from a ticket and insert an article link into my reply.
- As a **customer**, I browse and search public FAQs and articles in the portal.

## Acceptance criteria

- [ ] Article lifecycle: `Draft → Published → Archived`; publishing is a domain action, not a generic status setter.
- [ ] Visibility: `Internal` (agents only) vs `Public` (portal).
- [ ] Localized title/body per language; fallback language rules.
- [ ] Hierarchical categories.
- [ ] Search via PostgreSQL full-text (Arabic + English configs) behind a search abstraction.
- [ ] Article versioning / last-updated tracking.
- [ ] "Was this helpful?" feedback and view counts.
- [ ] Articles can be linked to tickets and ticket categories.

## Domain model

```text
KnowledgeArticle (aggregate: translations, status, visibility, versions)
KnowledgeCategory
ArticleFeedback
```

## Backend slices

```text
Features/KnowledgeBase/
├── Articles/ (Create, Update, Publish, Unpublish, Archive, GetById, List)
├── Categories/ (CRUD)
├── Faqs/
├── Search/
└── Feedback/
```

## Frontend

- Admin: article editor (rich text, AR/EN tabs), categories management.
- Agent: KB search panel in ticket details.
- Portal: public help center.

## Dependencies

- Platform (12) localization, Security (10). Feeds: Customer Portal (08), AI Features (07).
