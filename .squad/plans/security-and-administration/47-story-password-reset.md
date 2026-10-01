# Story 47 — Staff and portal customers can reset a forgotten password by email (Bug: BUG-11)

> Implemented in `customer-support-crm-api` commit `ba569b9` (`develop`, migration `20261001165819_AddPasswordReset`) and `customer-support-crm-web` commit `5ab9998` (`main`) on 2026-10-01. The plan was written against api `dca992c` / web `06c817a`; its line numbers refer to those commits. See *Implementation notes* at the end.
> Intake: [../../stories/security-and-administration/password-reset/intake.md](../../stories/security-and-administration/password-reset/intake.md)

## Prerequisites

- Story 02 — [02-story-authentication-jwt-and-refresh-tokens.md](02-story-authentication-jwt-and-refresh-tokens.md): `LoginHandler`, `UserSessionService`, change-password.
- Story 06 — [06-story-audit-logging.md](06-story-audit-logging.md): `IAuditTrail`, `AuditActions`.
- Story 30 — [../customer-portal/30-story-portal-accounts-and-authentication.md](../customer-portal/30-story-portal-accounts-and-authentication.md): `CustomerAccount`, portal register/verify/login.
- Story 41 — [../customer-portal/41-story-revoke-portal-sessions-on-access-revoke.md](../customer-portal/41-story-revoke-portal-sessions-on-access-revoke.md): `ActivePortalAccountFilter`.
- Story 32 — [../communication-channels/32-story-inbound-channels-and-outbound-messaging.md](../communication-channels/32-story-inbound-channels-and-outbound-messaging.md): `CustomerMessenger`, outbox dispatch.
- Frontend stories 09 and 17 — [../frontend/09-story-authentication-staff-and-portal.md](../frontend/09-story-authentication-staff-and-portal.md), [../frontend/17-story-customer-portal-ui.md](../frontend/17-story-customer-portal-ui.md).
- Related: Story 54 — [54-story-readable-audit-log.md](54-story-readable-audit-log.md) translates audit action codes. Whichever story lands second adds the en/ar labels for the two action codes introduced here.
- Source: manual QA report `.squad/qa/2026-10-01-manual-qa-report.md`, finding H1.
- Backend paths are relative to **`customer-support-crm-api/`**, frontend paths to **`customer-support-crm-web/`**.

---

## Story Goal

| | Before | After |
|---|---|---|
| Staff forgot their password | Only an administrator can reset it (`POST /users/{id}/reset-password`) | `/login` → "Forgot password?" → email link → `/login/reset-password?token=…` |
| Portal customer forgot their password | No way back in | `/portal/login` → "Forgot password?" → email link → `/portal/reset-password?token=…` |
| API | No forgot/reset endpoints (`grep -ri forgot src` finds nothing) | `POST /api/v1/auth/forgot-password`, `POST /api/v1/auth/reset-password`, `POST /api/v1/public/portal/forgot-password`, `POST /api/v1/public/portal/reset-password` |
| Help article "How to reset your password" | Points to a link that does not exist | The link exists and is named "Forgot password?" |

Rules for both account kinds:

1. **Request** (`forgot-password`): anonymous, `RateLimitPolicies.Authentication` (10 per minute per IP, `SecurityExtensions.cs` 159–169). It always returns 200 with the same message, whether the email is unknown, disabled, unverified or known, so it cannot be used to discover accounts (the same rule as `PortalRegisterHandler`, `PortalAccountSlices.cs` 76–79).
2. **Token:** 256 random bits from `ITokenService.GenerateRefreshToken()` (`TokenService.cs` 75–83: base64url text plus its SHA-256 hex hash). Only the hash is stored. It expires after **30 minutes**. A new request replaces the previous token, so only the latest link works. If a token was issued less than **60 seconds** ago, the request does nothing; this stops one address from being flooded with emails.
3. **Email:** queued in the outbox (`outbound_messages`) through `CustomerMessenger`, as `QueueVerificationCodeAsync` does (`CustomerMessaging.cs` 114–120). In **Development** the reset link is also logged, because the email channel is normally off (`appsettings.json` `Channels:Email:Enabled: false`) and `DispatchOutboxHandler` then marks the row `Failed` ("The Email channel is not configured.", `CustomerMessaging.cs` 251–255).
4. **Reset** (`reset-password`): token plus new password. The password must pass `ValidPassword` (`CommonRules.cs` 13–14, 12–128 characters). A used, expired, replaced or unknown token returns **400 `INVALID_RESET_TOKEN`** on field `token`. On success the password changes, the token is cleared, the lockout is cleared, sessions end (see below), and an audit record is written. The endpoint does not sign the user in; they sign in with the new password.
5. **Sessions:**
   - **Staff:** every active refresh token of the user is revoked with `RefreshTokenRevocationReason.PasswordChanged`, through `UserSessionService.RevokeAsync`, the same call the admin reset makes (`UserAdministration.cs` 101). Access tokens already issued stay valid until they expire (at most 15 minutes, `Jwt:AccessTokenLifetimeMinutes`), the same as for change-password today.
   - **Portal:** portal tokens are stateless 8-hour JWTs with no refresh token (`TokenService.cs` 42–53). The only per-request check is `ActivePortalAccountFilter` (`ActivePortalAccountFilter.cs` 16–35), which tests `IsActive`. A new `CustomerAccount.SessionsValidFrom` column is set at reset, and the filter rejects tokens whose `iat` is earlier.
6. **Audit:** `auth.password.reset_requested` / `auth.password.reset` (entity `User`) for staff, and `customers.portal_password_reset_requested` / `customers.portal_password_reset` (entity `Customer`, id = customer id) for portal accounts. The portal codes follow `customers.portal_access_granted` (`PortalAccountSlices.cs` 360). Requests are recorded only for accounts that receive an email. The audit values never contain the token, the link or the password (`AuditLog.cs` 6–9).

Disabled staff users (`!user.IsActive`) and portal accounts that are inactive or unverified get no email. Their reset tokens are rejected too, in case the account was disabled after the email was sent.

**Deviation from the intake:** none. Two design choices:

- **The token reuses the hash format of the portal verification code, not the code itself.** The intake said to reuse the portal register/verify pattern. That pattern stores a hash and an expiry as two columns on `CustomerAccount` (`VerificationCodeHash`, `VerificationExpiresAt`, `CustomerAccount.cs` 46–49), uses SHA-256 hex (`PortalSessions.HashCode`, `PortalAccountSlices.cs` 37), and keeps one live value per account. This plan keeps all of that. But the verification code is only a 6-digit number (`GenerateVerificationCode`, `CustomerAccount.cs` 104), and a link token can be longer, so it uses the 256-bit generator from refresh tokens instead.
- **The token is stored as two columns on `users` and `customer_accounts`, not in a new table.** There is one live token per account, so a separate table adds nothing.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Application/Features/Authentication/ChangePassword/ChangePasswordHandler.cs` (18–55) and `ChangePasswordEndpoint.cs` (12–24): the slice layout to copy (command, validator, handler, endpoint), and how sessions are revoked and the audit is recorded.
2. `src/CustomerSupportCrm.Application/Features/Authentication/Login/LoginEndpoint.cs` (15–25): `.AllowAnonymous()` inside the staff group plus `.RequireRateLimiting(RateLimitPolicies.Authentication)`. `LoginHandler.cs` 32–33: email normalization with `EmailAddress.Normalize`.
3. `src/CustomerSupportCrm.Application/Features/Authentication/Common/UserSessionService.cs` 48–60: `RevokeAsync`. `AuthenticationErrors.cs` 3–11: the error codes.
4. `src/CustomerSupportCrm.Application/Features/Users/Administration/UserAdministration.cs` 79–105: the admin reset (`ResetUserPasswordHandler`), the closest existing behavior. Its request contract is `ResetPasswordRequest(string NewPassword)` in `Contracts/Users/UserContracts.cs` 12. **Do not reuse or shadow that name**; the new contracts use other names.
5. `src/CustomerSupportCrm.Domain/Users/User.cs`: `ChangePasswordHash` (93–97), lockout (124–142), `IsActive` (69).
6. `src/CustomerSupportCrm.Domain/Customers/CustomerAccount.cs`: verification hash and expiry (46–49), `Verify` (106–124), `ChangePasswordHash` (126–130), lockout (132–149).
7. `src/CustomerSupportCrm.Application/Features/CustomerPortal/PortalAccountSlices.cs`: `PortalSessions.HashCode` (37), `PortalRegisterHandler` (80–120), `PortalResendVerificationHandler` (143–161, finds the customer's language), `PortalAuthEndpoints` (228–264, `IPublicEndpoint`, group `/portal`, mapped under `/api/v1/public` by `Api/Endpoints/EndpointExtensions.cs` 34–38).
8. `src/CustomerSupportCrm.Application/Features/CustomerPortal/ActivePortalAccountFilter.cs` 16–35.
9. `src/CustomerSupportCrm.Application/Features/Channels/CustomerMessaging.cs`: `CustomerTemplates.Build` (27–42) and `Verification` (57–60), `CustomerMessenger.QueueVerificationCodeAsync` (114–120), `DispatchOutboxHandler` (236–283).
10. `src/CustomerSupportCrm.Application/Abstractions/Channels/ChannelAbstractions.cs` 40–46: `CustomerPortalOptions` (`Portal:BaseUrl`, default `http://localhost:4200/portal`), bound in `Infrastructure/DependencyInjection.cs` 112. There is no setting for the staff web URL yet.
11. `src/CustomerSupportCrm.Infrastructure/Authentication/TokenService.cs` 42–53 (portal token, no refresh), 55–72 (`IssuedAt` is set, so the token carries `iat`), 75–83 (`GenerateRefreshToken`, `HashRefreshToken`). `Infrastructure/DependencyInjection.cs` 155: `MapInboundClaims = false`, so the claim type stays `iat`.
12. `src/CustomerSupportCrm.Domain/Audit/AuditActions.cs` 3–27, `Domain/Users/RefreshToken.cs` 5–11 (`RefreshTokenRevocationReason`).
13. `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/UserConfiguration.cs` 13–22 and `TicketConfiguration.cs` 100–116 (`CustomerAccountConfiguration`, `VerificationCodeHash` max length 128).
14. `src/CustomerSupportCrm.Application/Resources/Messages.resx` / `Messages.ar.resx`: `INVALID_CURRENT_PASSWORD` (69), `INVALID_VERIFICATION_CODE` (249 / 312).
15. Web: `src/app/features/auth/auth.routes.ts` (6–13), `login.page.ts` (60–112), `login.page.html` (145–169), `src/app/features/customer-portal/customer-portal.routes.ts` (14–17), `auth/portal-login.page.ts` (42–47), `auth/portal-verify.page.ts` (layout of a small anonymous portal form), `src/app/shared/password.ts` (`passwordValidators`, `matchFields`), `public/i18n/auth/en.json`, `public/i18n/portal/en.json` (`auth` at line 37).

---

## Backend Tasks

### 1 — Domain

**`User`** (`Domain/Users/User.cs`):

```csharp
public static readonly TimeSpan PasswordResetLifetime = TimeSpan.FromMinutes(30);
public static readonly TimeSpan PasswordResetCooldown = TimeSpan.FromMinutes(1);

/// <summary>SHA-256 of the emailed reset token; null when no reset is pending.</summary>
public string? PasswordResetTokenHash { get; private set; }
public DateTimeOffset? PasswordResetExpiresAt { get; private set; }

/// <summary>False when a token was issued less than <see cref="PasswordResetCooldown"/> ago.</summary>
public bool CanRequestPasswordReset(DateTimeOffset now) =>
    PasswordResetExpiresAt is not { } expires || expires - PasswordResetLifetime + PasswordResetCooldown <= now;

/// <summary>Replaces any pending token.</summary>
public void StartPasswordReset(string tokenHash, DateTimeOffset now)
{
    ArgumentException.ThrowIfNullOrWhiteSpace(tokenHash);
    PasswordResetTokenHash = tokenHash;
    PasswordResetExpiresAt = now + PasswordResetLifetime;
}

/// <summary>Single use: sets the password, clears the token and the lockout. False when the token is not valid.</summary>
public bool CompletePasswordReset(string tokenHash, string passwordHash, DateTimeOffset now)
{
    if (!IsActive || PasswordResetTokenHash is null || PasswordResetExpiresAt <= now
        || !CryptographicOperations.FixedTimeEquals(Encoding.UTF8.GetBytes(PasswordResetTokenHash), Encoding.UTF8.GetBytes(tokenHash)))
    {
        return false;
    }

    ChangePasswordHash(passwordHash);
    PasswordResetTokenHash = null;
    PasswordResetExpiresAt = null;
    FailedLoginAttempts = 0;
    LockoutEndsAt = null;
    return true;
}
```

**`CustomerAccount`** (`Domain/Customers/CustomerAccount.cs`): the same two properties, the constants, `CanRequestPasswordReset` and `StartPasswordReset`. Add:

```csharp
/// <summary>Portal tokens issued before this instant are rejected (see ActivePortalAccountFilter).</summary>
public DateTimeOffset? SessionsValidFrom { get; private set; }
```

`CompletePasswordReset` checks `IsActive && EmailVerified`. On success it also sets `SessionsValidFrom` to `now` truncated to whole seconds, because JWT `iat` is in whole seconds:

```csharp
SessionsValidFrom = DateTimeOffset.FromUnixTimeSeconds(now.ToUnixTimeSeconds());
```

A token issued earlier in that same second is still accepted. That gap is under one second and can be ignored.

The two entities share about 25 lines. That copying is accepted: it is how lockout is already copied between them (`User.cs` 124–142, `CustomerAccount.cs` 132–149).

### 2 — Persistence and migration

- `UserConfiguration`: `builder.Property(u => u.PasswordResetTokenHash).HasMaxLength(128);` and `builder.HasIndex(u => u.PasswordResetTokenHash).HasFilter("password_reset_token_hash IS NOT NULL");`.
- `CustomerAccountConfiguration` (`TicketConfiguration.cs` ~112): the same for `customer_accounts`. `SessionsValidFrom` needs no configuration.
- Migration (generate it **on top of the latest migration in `develop` when this story is implemented**; another story may add one first):

```
dotnet ef migrations add AddPasswordReset --project src/CustomerSupportCrm.Infrastructure --startup-project src/CustomerSupportCrm.Api --output-dir Persistence/Migrations
```

It adds three nullable columns: `users.password_reset_token_hash` (varchar 128), `users.password_reset_expires_at` (timestamptz) and the same two on `customer_accounts`. It also adds `customer_accounts.sessions_valid_from` (timestamptz) and two filtered indexes. No data changes. `Down` drops them.

### 3 — Configuration: staff web URL

The staff reset link needs the staff web app's base URL, and no setting holds it. Add next to `CustomerPortalOptions` in `Abstractions/Channels/ChannelAbstractions.cs`:

```csharp
public sealed class StaffAppOptions
{
    public const string SectionName = "StaffApp";

    /// <summary>Absolute base URL of the staff web app, e.g. https://support.example.com.</summary>
    public string BaseUrl { get; init; } = "http://localhost:4200";
}
```

Bind it in `Infrastructure/DependencyInjection.cs` `AddChannels` (after line 112), and add `"StaffApp": { "BaseUrl": "http://localhost:4200" }` to `appsettings.json` next to `"Portal"`. Links:

- staff: `{StaffApp:BaseUrl}/login/reset-password?token={Uri.EscapeDataString(token)}`
- portal: `{Portal:BaseUrl}/reset-password?token={…}`

### 4 — Email template and mailer

- `CustomerTemplates` (`CustomerMessaging.cs`): add

```csharp
public static (string Subject, string Heading, string Body, string Link) PasswordReset(string language) =>
    language == "ar"
        ? ("إعادة تعيين كلمة المرور", "إعادة تعيين كلمة المرور", "تلقّينا طلباً لإعادة تعيين كلمة المرور لحسابك. الرابط صالح لمدة 30 دقيقة ولمرة واحدة فقط. إذا لم تطلب ذلك فتجاهل هذه الرسالة؛ ستبقى كلمة مرورك كما هي.", "تعيين كلمة مرور جديدة")
        : ("Reset your password", "Reset your password", "We received a request to reset the password for your account. The link is valid for 30 minutes and can be used once. If you did not ask for this, ignore this email; your password stays the same.", "Set a new password");
```

- `CustomerMessenger`: add `QueuePasswordResetAsync(string email, string language, string resetUrl, CancellationToken)`. Model it on `QueueVerificationCodeAsync` (114–120): load the organization name and color, call `CustomerTemplates.Build(..., linkUrl: resetUrl, linkLabel: link)`, then `OutboundMessage.Queue(TicketChannel.Email, email, …, ticketId: null, messageId: null, now)`. It is used for staff too: the class only queues outbox rows.
- New `Features/Authentication/Common/PasswordResetMailer.cs` (`internal sealed class`, registered `AddScoped` in `Application/DependencyInjection.cs` next to `UserSessionService`, line 39), shared by the staff and portal handlers:

```csharp
internal sealed class PasswordResetMailer(
    CustomerMessenger messenger,
    IOptions<StaffAppOptions> staffApp,
    IOptions<CustomerPortalOptions> portal,
    IHostEnvironment environment,
    ILogger<PasswordResetMailer> logger)
{
    public Task QueueStaffAsync(string email, string token, CancellationToken ct) =>
        QueueAsync(email, CurrentLanguage(), $"{staffApp.Value.BaseUrl.TrimEnd('/')}/login/reset-password?token={Uri.EscapeDataString(token)}", ct);

    public Task QueuePortalAsync(string email, string language, string token, CancellationToken ct) =>
        QueueAsync(email, language, $"{portal.Value.BaseUrl.TrimEnd('/')}/reset-password?token={Uri.EscapeDataString(token)}", ct);

    private async Task QueueAsync(string email, string language, string url, CancellationToken ct)
    {
        await messenger.QueuePasswordResetAsync(email, language, url, ct);
        if (environment.IsDevelopment())
        {
            logger.LogInformation("Development only: password reset link for {Email}: {ResetUrl}", email, url);
        }
    }

    /// <summary>Staff users have no language setting; use the request culture (Accept-Language, UseRequestLocalization).</summary>
    private static string CurrentLanguage() => CultureInfo.CurrentUICulture.TwoLetterISOLanguageName == "ar" ? "ar" : "en";
}
```

This is the first `ILogger` in the Application project. `IHostEnvironment` and logging come from the existing `FrameworkReference Microsoft.AspNetCore.App` (`CustomerSupportCrm.Application.csproj`), so no package is added. For performance-analyzer rules (CA1848), use a `[LoggerMessage]` partial method if the build reports a warning.

The link also ends up in `outbound_messages.body` / `html_body`, as verification codes do. `GET /channels/outbox` does not return the body (`OutboundMessageResponse`, `CustomerMessaging.cs` 289), and the token is single-use and expires after 30 minutes.

### 5 — Contracts

- `Contracts/Authentication/AuthenticationContracts.cs` (the file holding `LoginRequest`): `public sealed record ForgotPasswordRequest(string Email);` and `public sealed record CompletePasswordResetRequest(string Token, string NewPassword);`.
- `Contracts/Portal/PortalContracts.cs`: reuse `PortalEmailRequest(string Email)` (line 9) for the request; add `public sealed record PortalResetPasswordRequest(string Token, string NewPassword);`.

### 6 — Error code and messages

- `AuthenticationErrors`: `public const string InvalidResetToken = "INVALID_RESET_TOKEN";`. The portal handlers use the same constant.
- `Messages.resx`: `INVALID_RESET_TOKEN` → "The reset link is invalid or has expired. Request a new one."
- `Messages.ar.resx`: `INVALID_RESET_TOKEN` → "رابط إعادة التعيين غير صالح أو منتهي الصلاحية. اطلب رابطاً جديداً."
- `AuditActions`: `PasswordResetRequested = "auth.password.reset_requested"`, `PasswordReset = "auth.password.reset"`. The portal codes are literal strings, as the other `customers.portal_*` codes are.

### 7 — Staff slices (`Features/Authentication/ForgotPassword/`, `Features/Authentication/ResetPassword/`)

Use the same layout as `ChangePassword/`: a command, a validator, a handler and an endpoint file each.

- `ForgotPasswordCommand(string Email) : IRequest`, validator `Email NotEmpty, MaximumLength(EmailAddress.MaxLength)`.
- `ForgotPasswordHandler(IApplicationDbContext db, ITokenService tokens, PasswordResetMailer mailer, IAuditTrail audit, TimeProvider time)`:

```csharp
var now = time.GetUtcNow();
var email = EmailAddress.Normalize(request.Email);
var user = await db.Users.SingleOrDefaultAsync(u => u.Email == email, ct);
if (user is null || !user.IsActive || !user.CanRequestPasswordReset(now))
{
    return; // Same response either way.
}

var token = tokens.GenerateRefreshToken();
user.StartPasswordReset(token.Hash, now);
await mailer.QueueStaffAsync(user.Email, token.Token, ct);
audit.Record(AuditActions.PasswordResetRequested, AuditEntityTypes.User, user.Id.ToString(), actorUserId: user.Id);
await db.SaveChangesAsync(ct);
```

- `ForgotPasswordEndpoint`: `POST {AuthenticationHttp.RoutePrefix}/forgot-password` → `ApiResults.Success("If the address belongs to an account, a password reset link has been sent.")`. Add `.AllowAnonymous()`, `.RequireRateLimiting(RateLimitPolicies.Authentication)`, `.WithName("ForgotPassword")`, `.WithTags("Authentication")` and `.Produces<ApiResponse<object?>>()`.
- `ResetPasswordCommand(string Token, string NewPassword) : IRequest`, validator `Token NotEmpty().MaximumLength(256)`, `NewPassword ValidPassword()`.
- `ResetPasswordHandler(IApplicationDbContext db, ITokenService tokens, IPasswordHasher hasher, UserSessionService sessions, IAuditTrail audit, IStringLocalizer<Messages> localizer, TimeProvider time)`:

```csharp
var hash = tokens.HashRefreshToken(request.Token.Trim());
var user = await db.Users.SingleOrDefaultAsync(u => u.PasswordResetTokenHash == hash, ct);
if (user is null || !user.CompletePasswordReset(hash, hasher.Hash(request.NewPassword), time.GetUtcNow()))
{
    throw new ValidationException([new ValidationFailure(nameof(request.Token), localizer[AuthenticationErrors.InvalidResetToken]) { ErrorCode = AuthenticationErrors.InvalidResetToken }]);
}

var userId = user.Id;
await sessions.RevokeAsync(t => t.UserId == userId, RefreshTokenRevocationReason.PasswordChanged, ct);
audit.Record(AuditActions.PasswordReset, AuditEntityTypes.User, userId.ToString(), actorUserId: userId);
await db.SaveChangesAsync(ct);
```

Hash the new password only after the token lookup succeeds; that avoids hashing for unknown tokens. Split the condition to do that.

- `ResetPasswordEndpoint`: `POST /auth/reset-password`, with the same attributes as above and `.WithName("ResetPassword")`. It returns `ApiResults.Success()`.

No CSRF header is needed: neither endpoint reads the refresh cookie (`AuthenticationHttp.RequireCsrfHeader` is for cookie-authenticated endpoints only).

### 8 — Portal slices (`PortalAccountSlices.cs`)

Add next to `PortalResendVerificationCommand` (141–161):

- `PortalForgotPasswordCommand(string Email) : IRequest` with `PortalForgotPasswordHandler`. It loads the account by normalized email and returns silently unless `account is { IsActive: true, EmailVerified: true }` and `CanRequestPasswordReset(now)`. The language is read as in line 157 (`Customers.PreferredLanguage`, default `"en"`). Then `StartPasswordReset`, `mailer.QueuePortalAsync`, and `audit.Record("customers.portal_password_reset_requested", "Customer", account.CustomerId.ToString())`, then `SaveChangesAsync`.
- `PortalResetPasswordCommand(string Token, string NewPassword) : IRequest` with a validator (`Token NotEmpty().MaximumLength(256)`, `NewPassword ValidPassword()`) and a handler. The handler looks the account up by `PasswordResetTokenHash`, calls `CompletePasswordReset`, and returns 400 `INVALID_RESET_TOKEN` on field `Token` on failure. On success it records `audit.Record("customers.portal_password_reset", "Customer", account.CustomerId.ToString())`. No customer timeline entry is added: `CustomerActivityTypes` (`Domain/Customers/CustomerRecords.cs` 118–127) has no fitting type, and this story does not add one.
- `PortalAuthEndpoints` (228–264): `group.MapPost("/forgot-password", (PortalEmailRequest …))` → the same "If the address belongs to an account…" success message, and `group.MapPost("/reset-password", (PortalResetPasswordRequest …))` → `ApiResults.Success()`. Both get `.RequireRateLimiting(RateLimitPolicies.Authentication)`, `.WithName("PortalForgotPassword")` / `"PortalResetPassword"` and `.Produces<ApiResponse<object?>>()`.

The actor of these audit rows is null: the caller is anonymous and `AuditTrail` uses the current staff user only when authenticated (`Infrastructure/Auditing/AuditTrail.cs` 28). The audit page shows them as "System", the same as other portal-originated entries.

### 9 — End older portal sessions (`ActivePortalAccountFilter.cs`)

Replace the `AnyAsync` in lines 25–27:

```csharp
var issuedAt = long.TryParse(context.HttpContext.User.FindFirst(JwtRegisteredClaimNames.Iat)?.Value, out var iat)
    ? DateTimeOffset.FromUnixTimeSeconds(iat)
    : DateTimeOffset.MinValue;

var active = await db.CustomerAccounts.AsNoTracking()
    .AnyAsync(a => a.Id == accountId && a.IsActive && (a.SessionsValidFrom == null || a.SessionsValidFrom <= issuedAt), ct);
```

It is still one primary-key lookup per portal request. Use `"iat"` as a constant if `JwtRegisteredClaimNames` (`Microsoft.IdentityModel.JsonWebTokens`) is not referenced by the Application project. Update the class summary: "…while its account is active and the token was issued after the last password reset".

### 10 — Help article

The article "How to reset your password" is **not seeded in code**. `grep -ri "reset your password\|forgot password"` over `customer-support-crm-api/src` finds nothing, and `Infrastructure/Persistence/Seed/DatabaseInitializer.cs` seeds no KB articles. It is data in the dev database (plan 29's verification step 4 creates such an article by hand). No code change. After the fix, its instruction "Click Forgot password" matches the new link text. Check the article in `/knowledge-base` and edit its text there if needed (see Verification step 9).

## Frontend Tasks

### Staff (`src/app/features/auth/`, i18n scope `auth`)

- `auth.routes.ts`: restructure `LOGIN_ROUTES` so the `/login` URL family gets two new children without touching `app.routes.ts`. Per the frontend overview (`plans/frontend/00-overview.md`, "Ownership"), `src/app/app.*` must not be edited.

```ts
export const LOGIN_ROUTES: Routes = [
  {
    path: '',
    resolve: { i18n: translationResolver('auth') },
    children: [
      { path: '', canActivate: [guestGuard], loadComponent: () => import('./login.page').then((m) => m.LoginPage) },
      { path: 'forgot-password', canActivate: [guestGuard], loadComponent: () => import('./forgot-password.page').then((m) => m.ForgotPasswordPage) },
      { path: 'reset-password', loadComponent: () => import('./reset-password.page').then((m) => m.ResetPasswordPage) },
    ],
  },
];
```

- `features/auth/password-reset.api.ts` (`@Injectable({ providedIn: 'root' })`): `forgot(email)` → `api.post<null>('/auth/forgot-password', { email }, { anonymous: true, silent: true })`, and `reset(token, newPassword)` → `api.post<null>('/auth/reset-password', { token, newPassword }, { anonymous: true, silent: true })`. `core/auth/auth.service.ts` is in `core/` and is not edited.
- `forgot-password.page.ts`: same card layout and `auth-pages.scss` as `login.page.html` (brand, language switcher, progress bar). One email field (`Validators.required, Validators.email, Validators.maxLength(256)`). On success, replace the form with a `sent` state showing `auth.forgot.sent` and a link back to `/login`. Validation errors go through `applyServerErrors`; other errors through `describeError`. A 429 from the rate limiter falls to `describeError`.
- `reset-password.page.ts`: read `token` from `route.snapshot.queryParamMap` once, keep it in a field, then drop it from the address bar with `router.navigate([], { relativeTo: route, queryParams: {}, replaceUrl: true })`. That keeps it out of history and of `Referer`. With no token, show `auth.errors.invalidResetToken` and a link to `/login/forgot-password`. Form: `newPassword` (`passwordValidators` from `shared/password.ts`) and `confirmPassword`, with `matchFields('newPassword', 'confirmPassword')` as in `profile.page.ts` 104–111. On success, show `auth.reset.done` with a "Sign in" button to `/login`. On `apiError.hasCode('INVALID_RESET_TOKEN')`, show `auth.errors.invalidResetToken` plus the "request a new link" link. Field errors on `newPassword` go through `applyServerErrors`.
- `login.page.html`: after the password `mat-form-field` (line 159), add `<a class="auth-card__forgot" routerLink="/login/forgot-password">{{ 'auth.login.forgotPassword' | t }}</a>`. Add `RouterLink` to the imports in `login.page.ts` (45–55) and a small right-aligned (`text-align: end`) rule in `auth-pages.scss`.

i18n `public/i18n/auth/en.json` / `ar.json` (same key set):

| Key | en | ar |
|---|---|---|
| `auth.login.forgotPassword` | Forgot password? | نسيت كلمة المرور؟ |
| `auth.forgot.title` | Reset your password | إعادة تعيين كلمة المرور |
| `auth.forgot.subtitle` | Enter your work email and we'll send you a link to choose a new password. | أدخل بريدك الإلكتروني للعمل وسنرسل إليك رابطاً لاختيار كلمة مرور جديدة. |
| `auth.forgot.submit` | Send reset link | إرسال رابط إعادة التعيين |
| `auth.forgot.sent` | If the address belongs to an account, we've sent a reset link. It expires in 30 minutes. | إذا كان العنوان مرتبطاً بحساب، فقد أرسلنا رابط إعادة التعيين. تنتهي صلاحيته خلال 30 دقيقة. |
| `auth.forgot.backToLogin` | Back to sign in | العودة إلى تسجيل الدخول |
| `auth.reset.title` | Choose a new password | اختر كلمة مرور جديدة |
| `auth.reset.subtitle` | Use at least 12 characters. You'll be signed out on every device. | استخدم 12 حرفاً على الأقل. سيتم تسجيل خروجك من جميع الأجهزة. |
| `auth.reset.submit` | Set new password | تعيين كلمة المرور |
| `auth.reset.done` | Your password has been reset. You can sign in now. | تمت إعادة تعيين كلمة المرور. يمكنك تسجيل الدخول الآن. |
| `auth.reset.signIn` | Sign in | تسجيل الدخول |
| `auth.reset.requestNew` | Request a new link | طلب رابط جديد |
| `auth.errors.invalidResetToken` | This reset link is invalid or has expired. | رابط إعادة التعيين غير صالح أو منتهي الصلاحية. |

### Portal (`src/app/features/customer-portal/`, i18n scope `portal`)

- `customer-portal.routes.ts` (after line 16): `{ path: 'forgot-password', canActivate: [portalGuestGuard], loadComponent: … PortalForgotPasswordPage }` and `{ path: 'reset-password', loadComponent: … PortalResetPasswordPage }`. Like `verify`, the reset route has no guard.
- `auth/portal-forgot-password.page.ts` and `auth/portal-reset-password.page.ts`: the same behavior as the staff pages, with the `crm-card portal-auth` layout and `portal-auth.scss` of `portal-login.page.ts`. API calls go through `ApiService` directly (`'/public/portal/forgot-password'`, `'/public/portal/reset-password'`, `{ anonymous: true, silent: true }`), as `portal-profile.page.ts` 139 does. `core/auth/portal-auth.service.ts` is not edited.
- `auth/portal-login.page.ts` `portal-auth__links` (42–47): add `<a routerLink="/portal/forgot-password">{{ 'portal.auth.forgotPassword' | t }}</a>`.

i18n `public/i18n/portal/en.json` / `ar.json`, under `auth`:

| Key | en | ar |
|---|---|---|
| `portal.auth.forgotPassword` | Forgot password? | نسيت كلمة المرور؟ |
| `portal.auth.forgotTitle` | Reset your password | إعادة تعيين كلمة المرور |
| `portal.auth.forgotSubtitle` | Enter the email you use to sign in and we'll send you a reset link. | أدخل البريد الإلكتروني الذي تسجّل الدخول به وسنرسل إليك رابط إعادة التعيين. |
| `portal.auth.sendResetLink` | Send reset link | إرسال رابط إعادة التعيين |
| `portal.auth.resetLinkSent` | If the address belongs to an account, we've sent a reset link. It expires in 30 minutes. | إذا كان العنوان مرتبطاً بحساب، فقد أرسلنا رابط إعادة التعيين. تنتهي صلاحيته خلال 30 دقيقة. |
| `portal.auth.resetTitle` | Choose a new password | اختر كلمة مرور جديدة |
| `portal.auth.resetSubtitle` | At least 12 characters. You'll be signed out on other devices. | 12 حرفاً على الأقل. سيتم تسجيل خروجك من الأجهزة الأخرى. |
| `portal.auth.resetSubmit` | Set new password | تعيين كلمة المرور |
| `portal.auth.resetDone` | Your password has been reset. You can sign in now. | تمت إعادة تعيين كلمة المرور. يمكنك تسجيل الدخول الآن. |
| `portal.auth.invalidResetToken` | This reset link is invalid or has expired. | رابط إعادة التعيين غير صالح أو منتهي الصلاحية. |
| `portal.auth.requestNewLink` | Request a new link | طلب رابط جديد |

`portal.auth.backToLogin` already exists and is reused.

---

## Test Plan

Test projects are **out of scope**: `tests/` must not be modified, and no e2e tests are added.

---

## Verification Steps

1. **Backend build:** `dotnet build src/CustomerSupportCrm.Api/CustomerSupportCrm.Api.csproj` gives 0 warnings and 0 errors. If a running API locks `bin/`, add `-o <temp dir>`.
2. **Migration:** `dotnet ef database update --project src/CustomerSupportCrm.Infrastructure --startup-project src/CustomerSupportCrm.Api` (or start the API in Development, which migrates on startup). `\d users` and `\d customer_accounts` show the new columns.
3. **Frontend build:** `cd customer-support-crm-web && npx ng build` gives 0 errors and 0 warnings.
4. **Enumeration:** `POST /api/v1/auth/forgot-password` with `admin@crm.local`, with an unknown address, and with a disabled user's address. Each returns 200 with the same message. Only the first adds an `outbound_messages` row (`GET /api/v1/channels/outbox`) and an `auth.password.reset_requested` audit entry.
5. **Dev log:** the API log shows "Development only: password reset link for admin@crm.local: http://localhost:4200/login/reset-password?token=…". With email off, the outbox row turns `Failed` ("The Email channel is not configured.").
6. **Staff reset:** sign in on a second browser first. Open the logged link, set a 12+ character password, sign in with it. In the second browser the next `POST /auth/refresh` fails (`INVALID_REFRESH_TOKEN`), so the session ends within 15 minutes. Open the same link again → "invalid or expired" (400 `INVALID_RESET_TOKEN`). An 11-character password → field error on the new password.
7. **Expiry, replacement, cooldown:** two requests within 60 seconds → only one outbox row. A request after 60 seconds → the first link is rejected and the second works. A link older than 30 minutes is rejected (or set `password_reset_expires_at` in the past in psql).
8. **Portal:** repeat steps 4–6 with a verified portal account on `/portal/login` → "Forgot password?". After the reset, a portal tab signed in before the reset gets 401 `ACCOUNT_DISABLED` on its next request. A new sign-in works. An unverified account gets no email.
9. **Lockout:** lock a staff account with 5 wrong passwords, then reset → you can sign in at once.
10. **UI and languages:** both sign-in pages show the link in en and ar (RTL). After opening the reset link, the address bar no longer shows the token. The public article `/help/articles/how-to-reset-your-password` (if present in the database) describes the "Forgot password?" link that now exists; edit its text in `/knowledge-base` if not.
11. **Audit:** `GET /api/v1/audit-logs?action=auth.password.reset` shows the reset; no entry contains the token or the link.

---

## Done Criteria

- [x] The four endpoints exist; both request endpoints answer 200 with the same message for any address.
- [x] Reset emails are queued in the outbox; in Development the link is logged.
- [x] Tokens are hashed, single-use, replaced by a newer request and expire after 30 minutes; bad tokens return 400 `INVALID_RESET_TOKEN` (en/ar messages).
- [x] The new password must pass `ValidPassword`; a reset clears the lockout.
- [x] A staff reset revokes every refresh token; a portal reset rejects portal tokens issued before it.
- [x] Requests and resets are audited without secrets.
- [x] Migration `AddPasswordReset` is generated by `dotnet ef` and applies cleanly.
- [x] "Forgot password?" links, request pages and reset pages exist for staff and portal, with en and ar keys.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` and `ng build` pass with zero warnings.

## Implementation notes (2026-10-01)

Built as planned, with these deviations:

| Deviation | Why |
|---|---|
| **No `ILogger` in the Application layer.** `PasswordResetMailer` only queues the email | Story 49 (done first) added the Development `Log` email provider, which writes the whole email, link included, to the API log. A second log line would only duplicate it |
| Domain split into `IsValidPasswordReset(hash, now)` and `CompletePasswordReset(passwordHash[, now])` instead of one `bool CompletePasswordReset(...)` | The plan asked to hash the new password only after the token check succeeds. The split makes that the natural order |
| The portal slices live in a new file, `Features/CustomerPortal/PortalPasswordResetSlices.cs` (own `IPublicEndpoint`, group `/portal`) | Keeps `PortalAccountSlices.cs` unchanged |
| The reset routes have no guard, staff and portal both (as `verify`) | A signed-in user may open an emailed link |

Verified at runtime (API plus the built-in browser; Chrome was disconnected for this run):
- Migration: the five columns exist after startup.
- Enumeration: a known address, an unknown address and a second request within 60 s all return 200 with the same message, and only **one** outbox row is added.
- Staff flow:
  - The link appears in the API log.
  - `/login` → "Forgot password?" → request page → "sent" state.
  - The reset page drops `?token` from the address bar. The client rejects "short" and mismatched passwords.
  - The reset succeeds. The pre-reset refresh cookie then gets `INVALID_REFRESH_TOKEN`, the old password `INVALID_CREDENTIALS`, and the new password works.
  - Reusing the link gives 400 `INVALID_RESET_TOKEN` on `token` (en/ar). An 11-character password gives `INVALID_LENGTH` on `newPassword`.
- Lockout: 5 wrong passwords → `ACCOUNT_LOCKED` → reset → sign-in succeeds at once.
- Portal flow:
  - A verified account gets the email. An unverified one gets the same response and no email.
  - Reset from `/portal/reset-password`. The pre-reset portal token gets 401 `ACCOUNT_DISABLED` (`sessions_valid_from` against `iat`), and a new sign-in works.
- Rate limit: the shared `authentication` policy (10/min per IP) returned 429 during the test burst, as designed.
- Audit: `auth.password.reset_requested` / `auth.password.reset` (User) and `customers.portal_password_reset_requested` / `customers.portal_password_reset` (Customer), with no token or link in the values.
- Arabic: "نسيت كلمة المرور؟" on both sign-in pages, RTL, and the request page title.

Story 54 still has to add en/ar labels for the four new audit action codes.

**STOP HERE. Report to the user.**
