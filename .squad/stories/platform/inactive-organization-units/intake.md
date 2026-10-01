# Story intake

- Folder: `.squad/stories/platform/inactive-organization-units/intake.md`

---

## Feature

- **Feature name (display):** Platform
- **Feature slug (folder under `plans/`):** `platform`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-06`
- **Work item type:** `Bug`
- **Status:** `To do`
- **Assignee:** ``
- **Labels:** `backend`, `bug`, `organization`

---

## Title

```
Reject assignment to inactive branches and departments
```

---

## Description

```
OrganizationErrors.InactiveUnit = "ORGANIZATION_UNIT_INACTIVE" (Common/Authorization/AccessScope.cs)
is defined but never thrown, so deactivated branches/departments can still be assigned to
customers, tickets, user scopes and categories.

Fix: every write that places data in a branch/department rejects inactive units with
422 ORGANIZATION_UNIT_INACTIVE (one shared helper next to EnsureCanAssign). Existing rows keep
their unit; reads are unchanged.
```

---

## Acceptance criteria

```
- [ ] Assigning an inactive branch or department returns 422 ORGANIZATION_UNIT_INACTIVE.
- [ ] Existing records in inactive units can still be read and updated without changing the unit.
- [ ] en/ar messages exist for the code.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** BUG-09 (resx sweep)
- **Depends on code areas or other stories:** found while writing the as-built plans 20–36 (see `.squad/HANDOFF.md`, "Probable backend bugs").

## Technical hints (optional)

- Repo root: `customer-support-crm-api/` (branch `develop`). .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
