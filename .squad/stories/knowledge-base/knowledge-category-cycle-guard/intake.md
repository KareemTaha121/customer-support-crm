# Story intake

- Folder: `.squad/stories/knowledge-base/knowledge-category-cycle-guard/intake.md`

---

## Feature

- **Feature name (display):** Knowledge Base
- **Feature slug (folder under `plans/`):** `knowledge-base`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-08`
- **Work item type:** `Bug`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `bug`, `kb`

---

## Title

```
Validate knowledge base category parents
```

---

## Description

```
KB category save (Features/KnowledgeBase/KnowledgeBaseSlices.cs ~140-146) rejects only direct
self-parenting and does not check that the parent exists.

Fix: the parent must exist (field error on parentId) and must not be the category or one of
its descendants (400 CATEGORY_CYCLE on parentId, en/ar).
```

---

## Acceptance criteria

```
- [ ] An unknown parent id is rejected.
- [ ] Setting a descendant as parent returns 400 CATEGORY_CYCLE.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** BUG-07 (same `CATEGORY_CYCLE` code and parent-chain check for ticket categories), BUG-09 (resx sweep)
- **Depends on code areas or other stories:** found while writing the as-built plans 20–36 (see `.squad/HANDOFF.md`, "Probable backend bugs").

## Technical hints (optional)

- Repo root: `customer-support-crm-api/` (branch `develop`). .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
