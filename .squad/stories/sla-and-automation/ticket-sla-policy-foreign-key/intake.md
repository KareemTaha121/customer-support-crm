# Story intake

- Folder: `.squad/stories/sla-and-automation/ticket-sla-policy-foreign-key/intake.md`

---

## Feature

- **Feature name (display):** SLA & Automation
- **Feature slug (folder under `plans/`):** `sla-and-automation`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-03`
- **Work item type:** `Bug`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `bug`, `sla`

---

## Title

```
Clear ticket SLA policy references when a policy is deleted
```

---

## Description

```
tickets.sla_policy_id has no foreign key to sla_policies. DeleteSlaPolicyHandler
(Features/Sla/SlaAdministration.cs ~127-138) says "the policy reference is cleared" but only
removes the policy, leaving tickets pointing at a deleted id.

Fix:
- Add an optional FK tickets.sla_policy_id -> sla_policies.id with ON DELETE SET NULL
  (EF configuration + migration; null out existing dangling ids in the migration first).
- Keep computed due dates on existing tickets (unchanged behaviour).
```

---

## Acceptance criteria

```
- [ ] Deleting an SLA policy sets sla_policy_id to null on its tickets; due dates are kept.
- [ ] The migration cleans existing dangling ids before adding the FK.
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
