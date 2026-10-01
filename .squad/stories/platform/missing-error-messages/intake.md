# Story intake

- Folder: `.squad/stories/platform/missing-error-messages/intake.md`

---

## Feature

- **Feature name (display):** Platform
- **Feature slug (folder under `plans/`):** `platform`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-09`
- **Work item type:** `Bug`
- **Status:** `To do`
- **Assignee:** ``
- **Labels:** `backend`, `bug`, `localization`

---

## Title

```
Add missing en/ar resx messages for error codes
```

---

## Description

```
These codes have no entry in Messages.resx or Messages.ar.resx (Application/Resources):
CHAT_CLOSED, NO_ACTIVE_BRANCH, OUTBOUND_MESSAGE_NOT_FOUND, WEBHOOK_DELIVERY_NOT_FOUND.
These exist only in Messages.ar.resx (English falls back to the exception text):
TICKET_CLOSED, INVALID_STATUS_TRANSITION, CATEGORY_NOT_FOUND.

Fix: add the missing keys in both files, and sweep for any other error-code constant
(UPPER_SNAKE under Application/ and Domain/) missing from either file.
```

---

## Acceptance criteria

```
- [ ] Every listed code has an en and an ar message.
- [ ] No error-code constant is missing from either resx file.
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
