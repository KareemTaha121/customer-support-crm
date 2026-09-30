# Customer Support CRM

This is the planning and workspace repository for the Customer Support CRM. The feature specs, stories, and implementation plans live here. The application code lives in two separate repositories.

| Repository | Role | Stack |
|---|---|---|
| **customer-support-crm** (this repo) | Planning, specs, and the squad-kit workspace | Markdown |
| [customer-support-crm-api](https://github.com/KareemTaha121/customer-support-crm-api) | Backend API | C# / .NET |
| [customer-support-crm-web](https://github.com/KareemTaha121/customer-support-crm-web) | Frontend web app | Angular / TypeScript |

## What's in this repo

```
.
├── .squad/                                       # squad-kit workspace (single source of truth)
│   ├── config.yaml
│   ├── features/                                 # product feature specs (01–12)
│   ├── stories/                                  # story intakes per feature
│   └── plans/                                    # generated implementation plans
├── customer-support-crm-implementation-plan.md   # overall implementation plan
├── customer-support-crm-api/                     # ← cloned separately, git-ignored here
└── customer-support-crm-web/                     # ← cloned separately, git-ignored here
```

Features and stories usually touch both the backend and the frontend, so they are kept in one place here instead of being copied into each code repo.

## Local setup

Clone this repo, then clone the two code repos inside it:

```bash
git clone https://github.com/KareemTaha121/customer-support-crm.git
cd customer-support-crm
git clone https://github.com/KareemTaha121/customer-support-crm-api.git
git clone https://github.com/KareemTaha121/customer-support-crm-web.git
```

The two child folders are listed in `.gitignore`, so each one keeps its own git history and remote. Commit and push from inside each folder:

- Planning changes go in this repo, from the root.
- Backend code goes in `customer-support-crm-api/`.
- Frontend code goes in `customer-support-crm-web/`.

See each code repo's `README.md` for build and run instructions.

## Workflow (squad-kit)

1. **Intake:** `squad new-story <feature-slug>` creates `.squad/stories/<feature>/<id>/intake.md`.
2. **Plan:** run `/squad-plan <intake-path>` in your agent.
3. **Implement:** open the code repo that the story targets (API or Web) and give the agent only the generated story file.

More details are in [.squad/README.md](.squad/README.md).
