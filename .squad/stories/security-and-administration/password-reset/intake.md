# Story intake

- Folder: `.squad/stories/security-and-administration/password-reset/intake.md`

---

## Feature

- **Feature name (display):** Security and Administration
- **Feature slug (folder under `plans/`):** `security-and-administration`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-11`
- **Work item type:** `Bug`
- **Status:** `To do`
- **Assignee:** ``
- **Labels:** `backend`, `frontend`, `bug`, `auth`, `portal`, `security`

---

## Title

```
Staff and portal customers can reset a forgotten password by email
```

---

## Description

```
Manual QA (2026-10-01, finding H1): there is no "forgot password" anywhere.
The spec requires it (.squad/features/10-security-and-administration.md line 64:
"Login, forgot/reset password."), but there is no endpoint, page or link on
/login (staff) or /portal/login (customers). The only ways back in today are an
administrator reset (POST /users/{id}/reset-password) for staff, and nothing at
all for portal customers.

The public help article "How to reset your password"
(/help/articles/how-to-reset-your-password) tells customers to "Click Forgot
password", but that link does not exist.

Fix:
- Staff: POST /api/v1/auth/forgot-password (anonymous, rate limited, always 200)
  and POST /api/v1/auth/reset-password (token + new password).
- Portal: the same pair under /api/v1/public/portal/.
- The token is random, single-use, stored only as a hash, and expires after
  30 minutes. A new request replaces the previous token.
- The reset email is queued through the existing outbox (outbound_messages),
  the same way portal verification codes are. In Development the reset link is
  also written to the log, because the email channel is usually not configured
  there and the outbox marks the message Failed.
- A successful reset uses the existing password policy (ValidPassword), clears
  the lockout, ends every staff session (refresh tokens) or portal session
  (tokens issued before the reset), and writes an audit record.
- Web: "Forgot password?" links on both sign-in pages, a request page and a
  reset page (token in the query string) for staff and for the portal, en + ar.
```

---

## Acceptance criteria

```
- [ ] POST /auth/forgot-password and /public/portal/forgot-password answer 200 with the same message for known, unknown and disabled addresses.
- [ ] A known active account gets an email in the outbox with a reset link; in Development the link is also logged.
- [ ] The link works once, within 30 minutes; a used, expired or replaced token returns 400 INVALID_RESET_TOKEN.
- [ ] The new password must pass ValidPassword (12-128 characters).
- [ ] After a staff reset, every refresh token of the user is revoked; after a portal reset, older portal tokens are rejected.
- [ ] A reset clears a login lockout.
- [ ] Requests and completed resets appear in the audit log; no token or password is stored in it.
- [ ] /login and /portal/login show a "Forgot password?" link; request and reset pages exist for both, in en and ar.
- [ ] `dotnet build` passes with zero warnings; `ng build` passes with zero errors and zero warnings.
```

---

## Attachments

None. Source: `.squad/qa/2026-10-01-manual-qa-report.md`, finding H1.

---

## Dependencies

- **Blocked by / related ids:** none. BUG-18 (plan 54, readable audit log) translates audit action codes; it should include the codes added here.
- **Depends on code areas or other stories:** staff auth (plan 02), portal accounts (plan 30), outbound messaging (plan 32), frontend auth (plan 09) and portal UI (plan 17).

## Technical hints (optional)

- Repo roots: `customer-support-crm-api/` (branch `develop`, .NET 10) and `customer-support-crm-web/` (branch `main`, Angular).
- Adds an EF Core migration (`AddPasswordReset`).

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- No e2e tests.
- Configuring a real SMTP provider.
- Changing the portal change-password endpoint (`POST /portal/me/change-password`), which still does not end other portal sessions.
- Access tokens already issued to staff stay valid until they expire (at most 15 minutes), as with change-password today.
