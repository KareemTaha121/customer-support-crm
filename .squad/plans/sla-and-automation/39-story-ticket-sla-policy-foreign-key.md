# Story 39 — Clear ticket SLA policy references when a policy is deleted (Bug: BUG-03)

> Fix plan, implemented in `customer-support-crm-api` commit `3433f13` (`develop`). Paths and line numbers refer to that commit.
> Intake: [../../stories/sla-and-automation/ticket-sla-policy-foreign-key/intake.md](../../stories/sla-and-automation/ticket-sla-policy-foreign-key/intake.md)

## Prerequisites

- Story 28 — [28-story-sla-policies-and-automation-engine.md](28-story-sla-policies-and-automation-engine.md): `SlaPolicy`, `DeleteSlaPolicyHandler`, the SLA fields on `Ticket`.
- Migration `20260930104602_AddSupportOperations` (`678ea67`) created `tickets.sla_policy_id` as a plain nullable `uuid` column with no foreign key.
- All paths are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

`DELETE /sla-policies/{id}` (`DeleteSlaPolicyHandler`, `Features/Sla/SlaAdministration.cs` 127–138) is documented as "Existing tickets keep their computed due dates; the policy reference is cleared". In practice it only removed the policy row. Tickets kept the deleted id, and `sla.policyName` read as null.

After the fix, the database clears the reference itself:

- `tickets.sla_policy_id` → `sla_policies.id` foreign key with `ON DELETE SET NULL`, plus an index on the column.
- Tickets keep `FirstResponseDueAt` / `ResolutionDueAt` and their breach flags, as before.
- The handler is unchanged; its doc comment is now true. `SlaPolicy` is hard-deleted (it is not `ISoftDeletable`), so the FK action fires.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/TicketConfiguration.cs`: the FK block (39–46). The other optional references (`CategoryId`, `AssignedAgentId`) already use `DeleteBehavior.SetNull`.
2. `src/CustomerSupportCrm.Domain/Sla/SlaPolicy.cs` line 11: `SlaPolicy : AggregateRoot<Guid>, IAuditableEntity` (no soft delete).
3. `src/CustomerSupportCrm.Domain/Tickets/Ticket.cs` line 79: `public Guid? SlaPolicyId { get; private set; }`.
4. `src/CustomerSupportCrm.Application/Features/Sla/SlaAdministration.cs` 127–138: `DeleteSlaPolicyHandler` (not modified).
5. `docs/development.md` lines 47–48: the `dotnet ef` commands.

---

## Backend Tasks

### 1 — Model

In `TicketConfiguration.cs`, add `using CustomerSupportCrm.Domain.Sla;` and, after the `AssignedAgentId` FK:

```csharp
// Deleting a policy clears the reference; tickets keep their computed due dates.
builder.HasOne<SlaPolicy>().WithMany().HasForeignKey(t => t.SlaPolicyId).OnDelete(DeleteBehavior.SetNull);
```

### 2 — Migration

Generate it:

```bash
dotnet ef migrations add AddTicketSlaPolicyForeignKey --project src/CustomerSupportCrm.Infrastructure --startup-project src/CustomerSupportCrm.Api --output-dir Persistence/Migrations
```

The command creates `Persistence/Migrations/20261001032735_AddTicketSlaPolicyForeignKey.cs` (+ `.Designer.cs`) and updates `ApplicationDbContextModelSnapshot.cs`. The snapshot changes only by the new index and FK.

In `Up`, **before** the generated `CreateIndex` / `AddForeignKey`, add a cleanup step. Policies deleted before this fix left dangling ids, and those would make `ADD CONSTRAINT` fail:

```sql
UPDATE tickets t
SET sla_policy_id = NULL
WHERE t.sla_policy_id IS NOT NULL
  AND NOT EXISTS (SELECT 1 FROM sla_policies p WHERE p.id = t.sla_policy_id);
```

`Down` drops the FK and the index; the cleared ids are not restored.

**No changes to:** domain, application handlers, contracts, resx, docs or the frontend. The SLA admin UI shows a null policy as "no policy".

---

## Test Plan

Test projects are **out of scope**: `tests/` must not be modified. No existing test covers SLA policies.

---

## Verification Steps

1. **Build:** in `customer-support-crm-api/` run `dotnet build`. At `3433f13`: 0 warnings, 0 errors.
2. **SQL (offline):**

   ```bash
   dotnet ef migrations script 20260930104602_AddSupportOperations AddTicketSlaPolicyForeignKey --project src/CustomerSupportCrm.Infrastructure --startup-project src/CustomerSupportCrm.Api
   ```

   Checked at `3433f13`. In one transaction, the script runs the `UPDATE` cleanup, `CREATE INDEX ix_tickets_sla_policy_id`, then `ALTER TABLE tickets ADD CONSTRAINT fk_tickets_sla_policies_sla_policy_id … ON DELETE SET NULL`.
3. **Apply:** `dotnet ef database update …` against the dev database. Applied on 2026-10-01 to the local dev database (`localhost:5432/customer_support_crm`): the cleanup `UPDATE`, `CREATE INDEX` and `ADD CONSTRAINT … ON DELETE SET NULL` ran, and `dotnet ef migrations list` shows nothing pending.
4. **Behaviour:** create policy P and a ticket that gets P (`GET /tickets/{id}` → `sla.policyId = P`). Then `DELETE /sla-policies/P` → 200; `GET /tickets/{id}` shows `sla.policyId: null` with the same due dates.
5. **Existing data:** on a database where a policy was deleted before the fix, the migration succeeds, and the affected tickets have `sla_policy_id IS NULL`.

---

## Done Criteria

- [x] Deleting an SLA policy sets `sla_policy_id` to null on its tickets; due dates are kept.
- [x] The migration cleans existing dangling ids before adding the FK.
- [x] The model snapshot matches the migration (generated by `dotnet ef`).
- [x] The migration has been applied to the local dev database (2026-10-01). Step 4 (delete a policy through the API) was not run.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to the next bug story.**
