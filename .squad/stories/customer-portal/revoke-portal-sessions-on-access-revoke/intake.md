# Story intake

- Folder: `.squad/stories/customer-portal/revoke-portal-sessions-on-access-revoke/intake.md`

---

## Feature

- **Feature name (display):** Customer Portal
- **Feature slug (folder under `plans/`):** `customer-portal`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-05`
- **Work item type:** `Bug`
- **Status:** `To do`
- **Assignee:** ``
- **Labels:** `backend`, `bug`, `portal`

---

## Title

```
Revoking portal access ends the customer's portal sessions
```

---

## Description

```
DELETE /customers/{id}/portal-access (RevokePortalAccessHandler, PortalAccountSlices.cs ~351-390)
disables the account, but issued customer access tokens stay valid for up to 8 hours.

Fix: portal requests reject tokens of disabled/revoked accounts (check account status per
request, like the staff GET /auth/me rule), returning 401.
```

---

## Acceptance criteria

```
- [ ] After revoke, the customer's existing token gets 401 on /portal/* immediately.
- [ ] Re-granting access lets the customer sign in again.
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
