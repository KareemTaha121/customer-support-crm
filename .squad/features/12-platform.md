# 12 — Platform

> **Source:** AZM Squad Customer Support CRM — Core Features §12
> **Implementation phase:** Phase 0 — Repository Foundation, Phase 1 — Backend Platform, Phase 3 — Organization Context
> **Status:** Done (backend + frontend) — backend plans [20–21, 42, 45](../plans/platform/00-overview.md), frontend plans [08](../plans/frontend/08-story-core-platform-shell.md), [19](../plans/frontend/19-story-administration-ui.md)
> **Build priority:** 1 (foundation — built first, used by every feature)

## Summary

Cross-cutting capabilities that make the CRM usable across AZM's languages, devices, departments, branches and brands.

## Scope

| Sub-feature | Description |
|-------------|-------------|
| Arabic & English | Full bilingual UI and API messages, RTL/LTR switching, localized dates/numbers. |
| Web and mobile friendly | Responsive Angular UI, accessible, works on phones/tablets. |
| Multi-department | Data and users scoped to departments. |
| Multi-branch | Data and users scoped to branches under an organization. |
| Custom branding | Logo, colors, name per organization (agent app, portal, emails). |

## User stories

- As a **user**, I switch between Arabic and English and the layout switches RTL/LTR.
- As a **user**, I can work comfortably from a mobile device.
- As an **admin**, I set up branches and departments and assign users to them.
- As an **admin**, I upload our logo and set brand colors used in the portal and emails.

## Acceptance criteria

- [ ] Backend foundation: PostgreSQL + EF Core, configuration, global exception handling, standard API response contract, correlation id, Serilog, health checks, OpenAPI, validation pipeline, API versioning.
- [ ] Localization: resource-based messages; `Accept-Language` honored; errors/validation localized.
- [ ] Frontend: Angular core/shared structure, interceptors (auth, refresh, correlation, localization, error), RTL/LTR, responsive layouts, accessibility basics.
- [ ] Organization → Branch → Department context model; every business entity declares ownership; no single-branch assumption.
- [ ] Branding settings applied via CSS variables/theme; used in portal and email templates.
- [ ] Soft delete and auditing conventions.
- [ ] Background jobs infrastructure.
- [ ] Docker, CI pipeline, environments, database migration strategy.

## Backend slices

```text
Features/Organization/ (shared with 10)
Features/Branding/ (Get, Update, UploadLogo)
Features/Localization/ (Languages, Resources)
Shared/ (Result, Errors, Paging, Behaviors, CurrentUser, Clock)
```

## Frontend

- `core/` (auth, interceptors, i18n, layout, theme), `shared/` (tables, forms, dialogs, components).
- Language switcher, responsive shell, branding theme loader.

## Dependencies

- None — required by all features.
