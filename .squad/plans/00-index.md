# Plans index

One row per feature folder under `.squad/plans/`. `NN` continues as a global execution sequence across all features when `naming.globalSequence` is `true` in `config.yaml`.

| Feature | Overview | NN range |
|---------|----------|----------|
| [security-and-administration](security-and-administration/00-overview.md) | Phase 2 — Identity & Authorization (users, roles, permissions, JWT, audit) | 01–07 |
| [frontend](frontend/00-overview.md) | Angular web app for all 12 features (staff app, customer portal, help center) | 08–19 |

Backend features 01–12 beyond Phase 2 were implemented directly on `customer-support-crm-api` `develop` (commits `65c74a3`…`678ea67`) without separate plan files.
