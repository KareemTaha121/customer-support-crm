# Story 45 — Add missing en/ar resx messages for error codes (Bug: BUG-09)

> Fix plan, implemented in `customer-support-crm-api` commit `5eb50e6` (`develop`). Paths refer to that commit.
> Intake: [../../stories/platform/missing-error-messages/intake.md](../../stories/platform/missing-error-messages/intake.md)

## Prerequisites

- Story 20 — [20-story-backend-foundation.md](20-story-backend-foundation.md): `ErrorResponseWriter.Localize`, `GlobalExceptionHandler`, the en/ar resx files.
- Commit `0936711` (feat(localization)) set the resx convention this story follows.
- Run after BUG-06, BUG-07 and BUG-08 (plans 42–44), which added `ORGANIZATION_UNIT_INACTIVE` (en), `BRANCH_NOT_FOUND` / `DEPARTMENT_NOT_FOUND` (en) and `CATEGORY_CYCLE` (en/ar).
- All paths are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

Every error code returned by the API should have an Arabic message, and an English one where English benefits from it.

**How localization works:** see `src/CustomerSupportCrm.Api/Middleware/ErrorResponseWriter.cs` lines 37–42. The handler calls `localizer[code]`. If the key is missing for the request culture, it returns the **throw-site message**, which is English.

**The convention from `0936711`:**

- `Messages.ar.resx` covers every business code.
- `Messages.resx` (English) gets only the codes with **one fixed English message**. A code with several or parameterized English messages stays out of it, so English responses keep the specific throw-site text.

**Deviation from the intake.** The intake listed `TICKET_CLOSED`, `INVALID_STATUS_TRANSITION` and `CATEGORY_NOT_FOUND` as missing in English. They are missing **on purpose**, because each one has several English messages:

| Code | English messages at throw sites |
|---|---|
| `TICKET_CLOSED` | "Closed tickets cannot be changed. Reopen it first." / "This ticket is closed. Please open a new request." |
| `INVALID_STATUS_TRANSITION` | "Only resolved or closed tickets can be reopened." / "Resolved tickets cannot be escalated; reopen first." / "Use escalation to escalate a ticket." |
| `CATEGORY_NOT_FOUND` | "The category was not found." / "The parent category was not found." |

Adding a generic English entry would make those responses less specific, so they were not added. The same applies to all the other ar-only codes (`INVALID_*` domain codes, `FILE_TOO_LARGE`, `DUPLICATE_CUSTOMER`, `UNKNOWN_SETTING`, …).

**What was actually missing.** Four codes had no entry in **either** file, so Arabic users got English text. Each has one fixed English message, so each now has an entry in both files:

| Code | Thrown at | en | ar |
|---|---|---|---|
| `CHAT_CLOSED` | `Domain/Tickets/ChannelRecords.cs` 208, 220 | The conversation is closed. | المحادثة مغلقة. |
| `NO_ACTIVE_BRANCH` | `Features/Channels/InboundChannels.cs` 75 | No active branch is configured. | لا يوجد فرع نشط مُعدّ. |
| `OUTBOUND_MESSAGE_NOT_FOUND` | `Features/Channels/CustomerMessaging.cs` 354 | The message was not found. | الرسالة غير موجودة. |
| `WEBHOOK_DELIVERY_NOT_FOUND` | `Features/Integrations/IntegrationSlices.cs` 162 | The delivery was not found. | عملية الإرسال غير موجودة. |

**Excluded by design:** the generic field codes `REQUIRED`, `INVALID`, `INVALID_LENGTH`, `INVALID_EMAIL`, `INVALID_FORMAT`, `OUT_OF_RANGE` and `INVALID_VALUE`. `GlobalExceptionHandler.GenericFieldCodes` keeps the validator's own message for them, because it names the field and the limit. The known gap that FluentValidation's Arabic text keeps the English property name is unchanged (HANDOFF, Known gaps).

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Api/Middleware/ErrorResponseWriter.cs` lines 37–42: `Localize`.
2. `src/CustomerSupportCrm.Api/Middleware/GlobalExceptionHandler.cs` lines 70–84: `ToApiError` and `GenericFieldCodes`.
3. `src/CustomerSupportCrm.Application/Resources/Messages.resx` and `Messages.ar.resx`. The new channel keys follow `CHAT_NOT_FOUND`; `WEBHOOK_DELIVERY_NOT_FOUND` follows `WEBHOOK_NOT_FOUND`.

---

## Backend Tasks

1. Add the four entries above to both resx files, next to the related keys. Keep each file's encoding and line endings.
2. **Sweep:** list every error code in `src/**/*.cs` (excluding `bin/`, `obj/` and `Migrations/`). That means every `const string X = "UPPER_SNAKE"`, every `new …Exception("UPPER_SNAKE", …)` and every `ErrorCode = "UPPER_SNAKE"`. Compare that list against the `<data name>` keys of both files.
3. **Result at `5eb50e6`:**
   - No code is missing from both files, apart from the generic field codes.
   - Every code missing only from English is a code with several English messages.
   - No key exists only in English.

**No changes to:** C# code, contracts, migrations, docs or the frontend.

---

## Test Plan

Test projects are **out of scope**: `tests/` must not be modified.

---

## Verification Steps

1. **Build:** `dotnet build src/CustomerSupportCrm.Api/CustomerSupportCrm.Api.csproj -o <temp dir>` at `5eb50e6` gives 0 warnings and 0 errors. The API was running locally and locking `bin/`.
2. **Sweep:** run the sweep from task 2 again. Its output is limited to the by-design lists above.
3. **Runtime:** with `Accept-Language: ar`:
   - `POST /api/v1/public/chat/conversations/{id}/messages` (header `X-Chat-Token`) on a closed chat → 422 `CHAT_CLOSED` "المحادثة مغلقة.".
   - `POST /api/v1/channels/outbox/{randomId}/retry` → 404 "الرسالة غير موجودة.".

---

## Done Criteria

- [x] Every code without a resx entry (`CHAT_CLOSED`, `NO_ACTIVE_BRANCH`, `OUTBOUND_MESSAGE_NOT_FOUND`, `WEBHOOK_DELIVERY_NOT_FOUND`) now has an en and an ar message.
- [x] No error-code constant is missing from `Messages.ar.resx`.
- [x] Codes missing from `Messages.resx` all have several English messages, which is the convention from `0936711`.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. All bug stories (37–45) are done; report to the user.**
