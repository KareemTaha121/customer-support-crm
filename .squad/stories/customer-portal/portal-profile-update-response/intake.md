# Story intake

- Folder: `.squad/stories/customer-portal/portal-profile-update-response/intake.md`

---

## Feature

- **Feature name (display):** Customer Portal
- **Feature slug (folder under `plans/`):** `customer-portal`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-04`
- **Work item type:** `Bug`
- **Status:** `To do`
- **Assignee:** ``
- **Labels:** `backend`, `bug`, `portal`

---

## Title

```
PUT /portal/me returns the updated name
```

---

## Description

```
PUT /portal/me (Features/CustomerPortal/PortalAccountSlices.cs ~277-288) updates
Customer.Name but returns PortalSessions.ProfileAsync, which reads CustomerAccount.DisplayName,
so the response (and a later GET /portal/me) still shows the old name.

Fix: update the account's display name together with the customer name (domain method on
CustomerAccount), in the same SaveChanges.
```

---

## Acceptance criteria

```
- [ ] The PUT /portal/me response and a following GET /portal/me show the new name.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** none
- **Depends on code areas or other stories:** found while writing the as-built plans 20–36 (see `.squad/HANDOFF.md`, "Probable backend bugs").

## Technical hints (optional)

- Repo root: `customer-support-crm-api/` (branch `develop`). .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
