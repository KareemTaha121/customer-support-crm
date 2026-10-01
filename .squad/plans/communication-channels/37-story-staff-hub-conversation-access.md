# Story 37 — Staff hub: check permission and scope before joining a chat conversation (Bug: BUG-01)

> Fix plan, implemented in `customer-support-crm-api` commit `d2563dc` (`develop`). Paths and line numbers refer to that commit.
> Intake: [../../stories/communication-channels/staff-hub-conversation-access/intake.md](../../stories/communication-channels/staff-hub-conversation-access/intake.md)

## Prerequisites

- Story 33 — [33-story-live-chat.md](33-story-live-chat.md): `ChatConversation`, `ChatHub`, `StaffHub`, `IChatAccessValidator`, `GetAgentChatMessagesHandler`.
- Story 21 — [../platform/21-story-organization-context-and-notifications.md](../platform/21-story-organization-context-and-notifications.md): `AccessScope`, `AccessScopeProvider`, `WhereInScope`, `RealtimeGroups`.
- All paths are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

`StaffHub.JoinConversation(conversationId)` used to add **any** signed-in staff connection to `RealtimeGroups.ChatConversation(id)`. A user without `chat.handle`, or outside the ticket's branch or department, could follow a conversation's live `chatMessage` / `chatUpdated` events if they knew its id. The HTTP routes (`/chat/conversations/*`) already checked both.

After the fix, a staff connection may join a conversation only when:

1. its token carries the `chat.handle` permission claim, and
2. the conversation's ticket is in the user's `AccessScope`, using exactly the rule of `GET /chat/conversations/{id}/messages`. `data.all_branches` means everything.

Otherwise the hub throws `HubException("Conversation not found.")`, the same message `ChatHub` uses, so the hub doesn't reveal whether the id exists. `LeaveConversation` is unchanged.

**Design note:** the scope is built from the hub connection's `ClaimsPrincipal` (`Context.User`) rather than through `ICurrentUser`/`IHttpContextAccessor`. Hub method invocations should not rely on the HTTP context accessor.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Infrastructure/Realtime/RealtimeHubs.cs` — `StaffHub` (17–58); `OnConnectedAsync` adds `chat.handle` holders to `chat-agents`; `JoinConversation` (36–51).
2. `src/CustomerSupportCrm.Infrastructure/Realtime/ChatHub.cs` — the visitor hub, the pattern followed (validator + `HubException`).
3. `src/CustomerSupportCrm.Application/Abstractions/Notifications/IChatAccessValidator.cs` — `CanJoinAsync` (visitor) and the new `CanStaffJoinAsync`.
4. `src/CustomerSupportCrm.Application/Common/Authorization/AccessScope.cs` — `AccessScopeProvider.GetAsync` (≈40–54) now delegates to `internal static LoadAsync(db, userId, allBranches, ct)` (60–79).
5. `src/CustomerSupportCrm.Application/Features/Channels/LiveChat.cs` — `AgentChats.FindTicketIdInScopeAsync` (69–80), `ChatAccessValidator` (83–103), `GetAgentChatMessagesHandler` (316–327).
6. `src/CustomerSupportCrm.Application/DependencyInjection.cs` line 56 — `IChatAccessValidator` → `ChatAccessValidator` (scoped; unchanged).

---

## Backend Tasks

### 1 — Scope loading without the HTTP context

In `AccessScope.cs`, extract the body of `AccessScopeProvider.GetAsync` into:

```csharp
internal static async Task<AccessScope> LoadAsync(IApplicationDbContext db, UserId userId, bool allBranches, CancellationToken cancellationToken)
```

`allBranches` returns `AccessScope.Everything`. Otherwise the method loads the user's `Scopes`: a branch-only row grants the branch, and a row with a department grants that department. `GetAsync` keeps its caching and its unauthenticated → empty-scope rule, then calls `LoadAsync(db, currentUser.UserId, currentUser.HasPermission(Permissions.DataAllBranches), ct)`. Add `using CustomerSupportCrm.Domain.Users;`.

### 2 — One shared conversation-in-scope lookup

In `LiveChat.cs`, add:

```csharp
internal static class AgentChats
{
    public static Task<Guid?> FindTicketIdInScopeAsync(IApplicationDbContext db, AccessScope scope, Guid conversationId, CancellationToken ct);
}
```

It runs the same query `GetAgentChatMessagesHandler` used inline: the conversation, joined to `db.Tickets.WhereInScope(scope)`, returning its `TicketId`, or null. `GetAgentChatMessagesHandler` now calls it and keeps throwing `NotFoundException(CHAT_NOT_FOUND)` on null.

### 3 — Validator

`IChatAccessValidator` gains:

```csharp
Task<bool> CanStaffJoinAsync(Guid conversationId, UserId userId, bool allBranches, CancellationToken cancellationToken);
```

`ChatAccessValidator.CanStaffJoinAsync` calls `AccessScopeProvider.LoadAsync`, then `AgentChats.FindTicketIdInScopeAsync(...) is not null`.

### 4 — Hub

`StaffHub` takes `IChatAccessValidator access` through its primary constructor. `JoinConversation` becomes `async` and throws `HubException("Conversation not found.")` when any of these holds:

- `Context.User` lacks the `CrmClaimTypes.Permission` = `Permissions.ChatHandle` claim;
- the `sub` claim is not a Guid;
- `CanStaffJoinAsync(conversationId, new UserId(sub), hasClaim(data.all_branches), Context.ConnectionAborted)` returns false.

Otherwise it adds the connection to `RealtimeGroups.ChatConversation(conversationId)` as before.

### 5 — Docs

`docs/endpoints.md` (realtime table, `/hubs/staff` row): `JoinConversation(id)` now says *(needs `chat.handle` and the ticket in scope)*.

**No changes to:** contracts, migrations, resx (a hub error is not localized; same as `ChatHub`), and the frontend. `chat-console.page.ts` already ignores a failed `JoinConversation` (`.catch(() => undefined)`), and its HTTP calls return 404 for out-of-scope chats anyway.

---

## Test Plan

Test projects are **out of scope**: `tests/` must not be modified. No existing test references `StaffHub`, `JoinConversation` or `IChatAccessValidator`, so no test double needs the new interface member.

---

## Verification Steps

1. **Build:** in `customer-support-crm-api/` run `dotnet build`. Result at `d2563dc`: 0 warnings, 0 errors.
2. **In scope:** sign in as a chat agent scoped to the chat's branch. Connect a SignalR client to `/hubs/staff?access_token=$TOKEN` and invoke `JoinConversation(CID)`; it succeeds. Post a visitor message, and the client receives `chatMessage`.
3. **Out of scope:** use an agent with `chat.handle` scoped to another branch. `JoinConversation(CID)` fails with `HubException` "Conversation not found." and no `chatMessage` arrives.
4. **No permission:** use a staff user without `chat.handle`. `JoinConversation(CID)` fails the same way.
5. **All branches:** an admin with `data.all_branches` joins any conversation.
6. **Unknown id:** a random Guid fails the same way as case 3.
7. **HTTP parity:** for the users in cases 3 and 4, `GET /api/v1/chat/conversations/CID/messages` returns 404 or 403. That is the same decision as the hub's.

---

## Done Criteria

- [x] A staff connection without `chat.handle` cannot join any conversation.
- [x] A chat agent cannot join a conversation whose ticket is outside their branch/department scope.
- [x] In-scope agents and `data.all_branches` users join as before; the chat console is unchanged.
- [x] The hub and `GET /chat/conversations/{id}/messages` share `AgentChats.FindTicketIdInScopeAsync`.
- [x] The scope is built from the hub's claims, not `IHttpContextAccessor`.
- [x] `docs/endpoints.md` is updated.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to the next bug story.**
