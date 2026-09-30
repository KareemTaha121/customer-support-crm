# Story intake

- Folder: `.squad/stories/security-and-administration/audit-logging/intake.md`

---

## Feature

- **Feature name (display):** Security & Administration — Phase 2 Identity & Authorization
- **Feature slug (folder under `plans/`):** `security-and-administration`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `P2-06`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `phase-2`, `backend`, `audit`

---

## Title

```
Append-only audit log for entity changes and security events
```

---

## Description

```
Record who did what, when and from where (customer-support-crm-api).

Domain (Domain/Audit/AuditLog):
- Id, OccurredAt, ActorUserId (nullable), ActorEmail, Action, EntityType,
  EntityId, OldValues (jsonb), NewValues (jsonb), CorrelationId, IpAddress,
  UserAgent. Created only through a factory; no mutators.
- Actions: Created, Updated, Deleted, LoginSucceeded, LoginFailed,
  AccountLocked, Logout, RefreshTokenReuseDetected, PasswordChanged,
  RolesChanged, PermissionsChanged, UserActivated, UserDeactivated.

Request context (Application/Abstractions):
- IRequestContext: CorrelationId, IpAddress, UserAgent — implemented in Api from
  HttpContext (reuse the correlation id helpers from Phase 1).

Entity change auditing (Infrastructure/Persistence/Interceptors):
- AuditSaveChangesInterceptor captures Added/Modified/Deleted entries for
  entities implementing IAuditable (User, Role, and later business entities),
  writes AuditLog rows in the same SaveChanges transaction.
- Modified entries store only changed properties (old/new).
- Redaction: properties marked [AuditIgnore] or in a deny-list are never
  written: PasswordHash, SecurityStamp, TokenHash, and any RefreshToken entity.
- Join changes (UserRole, RolePermission) are recorded as RolesChanged /
  PermissionsChanged on the parent with before/after sets.

Append-only:
- ApplicationDbContext rejects Modified or Deleted AuditLog entries (throw
  InvalidOperationException); no update/delete endpoints exist.

Security events:
- IAuditLogger.LogSecurityEventAsync(action, userId?, email?, details?) used by
  the authentication handlers from P2-02 (login success/failure, lockout,
  logout, refresh-token reuse) and the password-change / activate / deactivate
  handlers from P2-04. Failed-login events must be persisted even though the
  login request returns 401.

Search slice (Application/Features/AuditLogs/Search):
- GET /api/v1/audit-logs, permission audit.view. Filters: actorUserId,
  entityType, entityId, action, from, to; paged (pageSize max 100), sorted by
  OccurredAt desc by default. Index audit_logs(occurred_at), (entity_type,
  entity_id), (actor_user_id).
- Add EF migration `AddAuditLog`.
```

---

## Acceptance criteria

```
- [ ] Creating/updating a user or role writes AuditLog rows with correct old/new values and correlation id.
- [ ] Password hash, security stamp and refresh tokens never appear in any audit row.
- [ ] Login success/failure, lockout, logout and refresh-token reuse are audited with IP and user agent.
- [ ] Attempting to modify or delete an AuditLog through the DbContext throws.
- [ ] GET /api/v1/audit-logs filters, pages and requires audit.view.
- [ ] `AddAuditLog` migration applies cleanly.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** P2-01, P2-02, P2-03, P2-04, P2-05
- **Depends on code areas or other stories:** ApplicationDbContext, CorrelationIdMiddleware, ICurrentUser, auth and user handlers.

## Extra notes (optional)

- Implementation plan §13 (auditing, never-log list), §25 (correlation).

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10, Npgsql jsonb columns.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- Audit export (CSV/Excel) and retention policies (Phase 3).
- Auditing read/view operations.
