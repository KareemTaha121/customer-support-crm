# Security & Administration — Phase 2 Identity & Authorization

Feature spec: [../../features/10-security-and-administration.md](../../features/10-security-and-administration.md) (Phase 2 section).
Repo: `customer-support-crm-api`. Out of scope for every story: Docker, deployment, CI/CD, and files under `tests/`.

| NN | Tracker id | Plan file | Title | Depends on | Status |
|----|-----------|-----------|-------|-----------|--------|
| 01 | P2-01 | [01-story-identity-domain-persistence.md](01-story-identity-domain-persistence.md) | Identity domain & persistence | Phase 1 | Done |
| 02 | P2-02 | — | Authentication: JWT + rotating refresh tokens | 01 | Done |
| 03 | P2-03 | — | Current user & permission authorization | 01, 02 | Done |
| 04 | P2-04 | — | User management | 01–03 | Done |
| 05 | P2-05 | — | Role & permission management | 01, 03 | Done |
| 06 | P2-06 | — | Audit logging | 01–05 | Done |
| 07 | P2-07 | — | Security hardening | 02, 03 | Done |

Plans are generated one at a time, after the previous story is implemented, so each plan cites the real code the earlier stories produced.

Note: `IPasswordHasher` moved from P2-02 into Story 01 because the bootstrap-admin seeder needs it.

All seven stories were implemented in `customer-support-crm-api` commit `0f87e2d` (feat: add identity, roles, permissions and audit logging). Only Story 01 has a plan file; 02–07 were implemented straight from their intakes.
