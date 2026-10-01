# Story intake

- Folder: `.squad/stories/ticket-management/ticket-category-cycle-guard/intake.md`

---

## Feature

- **Feature name (display):** Ticket Management
- **Feature slug (folder under `plans/`):** `ticket-management`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-07`
- **Work item type:** `Bug`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `bug`, `tickets`

---

## Title

```
Prevent cycles in ticket category parents
```

---

## Description

```
Ticket category save (Features/Tickets/TicketCategorySlices.cs ~57-72) only checks that the
parent exists, so A -> B -> A cycles are possible.

Fix: walk the parent chain before saving; reject a parent that is the category or one of its
descendants with a 400 field error on parentId (new code CATEGORY_CYCLE, en/ar).
```

---

## Acceptance criteria

```
- [ ] Setting the category itself or a descendant as parent returns 400 CATEGORY_CYCLE on parentId.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** BUG-08 (same `CATEGORY_CYCLE` code and parent-chain check for KB categories), BUG-09 (resx sweep)
- **Depends on code areas or other stories:** found while writing the as-built plans 20–36 (see `.squad/HANDOFF.md`, "Probable backend bugs").

## Technical hints (optional)

- Repo root: `customer-support-crm-api/` (branch `develop`). .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
