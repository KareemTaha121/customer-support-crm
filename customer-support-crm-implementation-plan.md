# Customer Support CRM — Production-Ready Technical Implementation Plan

> **Purpose:** This document is the implementation source of truth for building the Customer Support CRM from scratch.
>
> **Repositories:** `customer-support-crm-api` and `customer-support-crm-web`
>
> **Backend:** Latest stable .NET / ASP.NET Core at implementation time
>
> **Frontend:** Latest stable Angular at implementation time
>
> **Database:** PostgreSQL
>
> **Architecture:** Vertical Slice + DDD + Clean Architecture boundaries + CQRS
>
> **Status:** Architecture and implementation blueprint only. Do not generate application code from this document until the implementation phases are approved.

---

## 1. Product Scope

The CRM is a customer-support platform covering:

1. Customer Management
2. Ticket Management
3. Communication Channels
4. Agent Dashboard
5. SLA & Automation
6. Knowledge Base
7. AI Features
8. Customer Portal
9. Reports & Management
10. Security & Administration
11. Integrations
12. Platform capabilities

The supplied product requirements explicitly include customer profiles, contact details, interaction history, notes and attachments; ticket creation/tracking, categories, priorities, assignment, status, escalation and history; Email, WhatsApp, live chat, SMS and web forms; agent dashboards, tasks/reminders, quick replies and collaboration; SLA targets, automatic assignment, escalation and notifications; FAQs/help articles/solutions/search; AI summaries, suggested replies, categorization, suggested solutions and chatbot; customer portal capabilities; reporting; users/roles/permissions/audit logs; integrations; Arabic/English, responsive web/mobile-friendly UI, multi-department, multi-branch and custom branding. fileciteturn0file0L2-L35 fileciteturn0file0L37-L70

### 1.1 Initial implementation priorities

Build in this order:

1. Platform foundation
2. Authentication / users / roles / permissions
3. Organizations/branches/departments and tenant/workspace boundary
4. Customers
5. Tickets
6. Ticket assignment/status/history
7. Notes and attachments
8. Agent dashboard
9. SLA
10. Notifications
11. Knowledge base
12. Customer portal
13. Communication adapters
14. Reports
15. AI features
16. Advanced automation/integrations

The architecture must allow later features to be added as independent vertical slices without restructuring the entire application.

---

# 2. Architecture Overview

## 2.1 Architectural decision

Use a **Vertical Slice Architecture inside a Clean Architecture dependency boundary**.

Do **not** build a classic:

```text
Controllers/
Services/
Repositories/
DTOs/
Validators/
```

application where all features are mixed together.

Instead:

```text
Feature
 ├── Command
 │    ├── Endpoint
 │    ├── Command
 │    ├── Handler
 │    ├── Validator
 │    └── Response
 ├── Query
 │    ├── Endpoint
 │    ├── Query
 │    ├── Handler
 │    └── Response
 └── ...
```

The business domain remains protected from infrastructure concerns.

### 2.2 Dependency direction

```text
API
 │
 ▼
Application / Features
 │
 ▼
Domain

Infrastructure ───────► Application
Infrastructure ───────► Domain
API ───────────────────► Application
API ───────────────────► Infrastructure
```

The important rule is:

```text
Domain
  └── depends on nothing application-specific or infrastructure-specific
```

Infrastructure implements technical concerns required by Application/Domain.

### 2.3 Practical solution structure

```text
CustomerSupportCrm.sln

src/
  CustomerSupportCrm.Api/
  CustomerSupportCrm.Application/
  CustomerSupportCrm.Domain/
  CustomerSupportCrm.Infrastructure/
  CustomerSupportCrm.Contracts/

tests/
  CustomerSupportCrm.Domain.Tests/
  CustomerSupportCrm.Application.Tests/
  CustomerSupportCrm.IntegrationTests/
  CustomerSupportCrm.Api.Tests/
```

The application layer is not organized by generic technical folders. It is organized by feature.

---

# 3. Backend Repository

Repository:

```text
customer-support-crm-api/
```

Recommended top-level structure:

```text
customer-support-crm-api/
├── src/
│   ├── CustomerSupportCrm.Api/
│   ├── CustomerSupportCrm.Application/
│   ├── CustomerSupportCrm.Domain/
│   ├── CustomerSupportCrm.Infrastructure/
│   └── CustomerSupportCrm.Contracts/
│
├── tests/
│   ├── CustomerSupportCrm.Domain.Tests/
│   ├── CustomerSupportCrm.Application.Tests/
│   ├── CustomerSupportCrm.IntegrationTests/
│   └── CustomerSupportCrm.Api.Tests/
│
├── docs/
│   ├── architecture.md
│   ├── api-contract.md
│   ├── security.md
│   └── development.md
│
├── deploy/
│   ├── docker/
│   └── kubernetes/
│
├── .editorconfig
├── .gitignore
├── Directory.Build.props
├── Directory.Packages.props
├── global.json
├── docker-compose.yml
├── README.md
└── CustomerSupportCrm.sln
```

---

# 4. Backend Domain Layer

## 4.1 Domain responsibilities

The Domain owns:

- Entities
- Aggregates
- Value Objects
- Domain events
- Domain services
- Business rules
- Invariants
- Enumerations where domain semantics require them

The Domain must not know about:

- EF Core
- PostgreSQL
- ASP.NET Core
- MediatR
- JWT
- Serilog
- HTTP
- Email providers
- WhatsApp providers
- Angular

## 4.2 Domain structure

```text
CustomerSupportCrm.Domain/
├── Common/
│   ├── Entity.cs
│   ├── AggregateRoot.cs
│   ├── DomainEvent.cs
│   ├── IDomainEvent.cs
│   ├── IAuditableEntity.cs
│   └── ISoftDeletable.cs
│
├── Customers/
│   ├── Customer.cs
│   ├── CustomerId.cs
│   ├── CustomerStatus.cs
│   ├── CustomerCreatedDomainEvent.cs
│   └── ValueObjects/
│       ├── CustomerName.cs
│       ├── EmailAddress.cs
│       └── PhoneNumber.cs
│
├── Tickets/
│   ├── Ticket.cs
│   ├── TicketId.cs
│   ├── TicketStatus.cs
│   ├── TicketPriority.cs
│   ├── TicketNumber.cs
│   ├── TicketMessage.cs
│   ├── TicketAssignedDomainEvent.cs
│   ├── TicketStatusChangedDomainEvent.cs
│   └── ValueObjects/
│
├── KnowledgeBase/
├── Sla/
├── Users/
├── Organizations/
├── Departments/
├── Notifications/
├── Attachments/
├── Audit/
└── Shared/
```

Use domain terminology from the product instead of technical terminology.

---

# 5. DDD Guidelines

## 5.1 Entities

Entities have identity and lifecycle.

Examples:

- Customer
- Ticket
- TicketMessage
- KnowledgeArticle
- User
- Department
- Branch

Do not create an anemic entity containing only public setters.

Prefer:

```text
Ticket.AssignTo(agent)
Ticket.ChangePriority(priority)
Ticket.ChangeStatus(status)
Ticket.Escalate(reason)
Ticket.AddMessage(...)
```

over:

```text
ticket.Status = ...
ticket.AssignedAgentId = ...
```

The entity should protect its invariants.

## 5.2 Aggregates

Choose aggregates based on transactional consistency.

Initial aggregate candidates:

```text
Customer
Ticket
KnowledgeArticle
SlaPolicy
```

Do not make every database table an aggregate.

A Ticket aggregate should control operations that must remain consistent within the ticket boundary.

## 5.3 Value Objects

Use value objects for concepts whose identity is their value.

Examples:

- EmailAddress
- PhoneNumber
- CustomerName
- TicketNumber
- Address
- Money
- DateRange

Avoid primitive obsession where domain behavior benefits from stronger types.

## 5.4 Domain events

Use domain events for business facts:

```text
TicketCreated
TicketAssigned
TicketStatusChanged
TicketEscalated
SlaBreached
CustomerCreated
```

Domain events must represent facts, not commands.

Handlers for side effects such as notifications, audit entries or integrations belong outside the domain.

---

# 6. Application Layer

The Application project contains feature slices and application orchestration.

```text
CustomerSupportCrm.Application/
├── Abstractions/
│   ├── Authentication/
│   ├── Authorization/
│   ├── Persistence/
│   ├── Clock/
│   ├── Messaging/
│   ├── Files/
│   ├── Notifications/
│   ├── Localization/
│   └── Integrations/
│
├── Behaviors/
│   ├── ValidationBehavior.cs
│   ├── LoggingBehavior.cs
│   ├── TransactionBehavior.cs
│   └── AuthorizationBehavior.cs
│
├── Features/
│   ├── Authentication/
│   ├── Customers/
│   ├── Tickets/
│   ├── KnowledgeBase/
│   ├── Sla/
│   ├── Notifications/
│   ├── Dashboard/
│   ├── Reports/
│   ├── Users/
│   ├── Roles/
│   ├── Permissions/
│   ├── Departments/
│   ├── Branches/
│   ├── Attachments/
│   ├── CustomerPortal/
│   ├── Integrations/
│   └── Ai/
│
└── DependencyInjection.cs
```

---

# 7. Vertical Slice Pattern

Every use case should be self-contained.

Example:

```text
Features/
└── Tickets/
    └── Create/
        ├── CreateTicketCommand.cs
        ├── CreateTicketValidator.cs
        ├── CreateTicketHandler.cs
        ├── CreateTicketEndpoint.cs
        └── CreateTicketResponse.cs
```

A larger slice may contain:

```text
Features/
└── Tickets/
    └── GetById/
        ├── GetTicketByIdQuery.cs
        ├── GetTicketByIdHandler.cs
        ├── GetTicketByIdResponse.cs
        └── GetTicketByIdEndpoint.cs
```

Do not create a global `TicketService`.

If multiple slices genuinely share business behavior, move the behavior into the Domain or a clearly justified Application abstraction.

---

# 8. CQRS

## 8.1 Commands

Commands change state.

Examples:

```text
CreateTicket
AssignTicket
ChangeTicketStatus
AddTicketMessage
EscalateTicket
CreateCustomer
UpdateCustomer
```

Rules:

- Commands express intent.
- Command handlers orchestrate the use case.
- Validation happens before the handler executes.
- Domain rules remain in entities/aggregates.
- Return a meaningful result/response.

## 8.2 Queries

Queries read data.

Examples:

```text
GetTicketById
SearchTickets
GetCustomerById
SearchCustomers
GetAgentDashboard
GetSlaPerformanceReport
```

Queries should use read-optimized projections.

Do not load a complete aggregate if a query only needs five fields.

For complex reporting queries, direct EF Core projections are preferred over generic repositories.

## 8.3 MediatR

Use MediatR for:

- Commands
- Queries
- Domain/application notifications
- Pipeline behaviors

Do not use MediatR merely to add indirection to trivial code.

---

# 9. Infrastructure Layer

```text
CustomerSupportCrm.Infrastructure/
├── Persistence/
│   ├── ApplicationDbContext.cs
│   ├── Configurations/
│   │   ├── CustomerConfiguration.cs
│   │   ├── TicketConfiguration.cs
│   │   └── ...
│   ├── Migrations/
│   ├── Interceptors/
│   │   ├── AuditingInterceptor.cs
│   │   └── SoftDeleteInterceptor.cs
│   └── Seed/
│
├── Authentication/
├── Authorization/
├── Identity/
├── Files/
├── Notifications/
├── Email/
├── Sms/
├── WhatsApp/
├── Search/
├── Ai/
├── BackgroundJobs/
├── Integrations/
├── Logging/
├── Time/
└── DependencyInjection.cs
```

Infrastructure owns implementation details.

---

# 10. EF Core + PostgreSQL

Use:

- PostgreSQL
- EF Core PostgreSQL provider
- Code-first migrations
- Explicit entity configurations
- `IEntityTypeConfiguration<T>`
- Appropriate indexes
- UTC timestamps
- Optimistic concurrency where useful

Avoid putting persistence annotations into Domain entities.

Example configuration location:

```text
Infrastructure/
└── Persistence/
    └── Configurations/
```

## 10.1 DbContext

Use a single application DbContext initially unless scaling requirements demonstrate a need for separation.

The DbContext is an infrastructure concern.

Do not expose EF Core types through the API contract.

## 10.2 Repositories

Do not create:

```text
IGenericRepository<T>
GenericRepository<T>
```

by default.

Use EF Core directly from feature handlers for simple application queries/commands.

Create a repository or domain-specific persistence abstraction only when it provides meaningful value, such as:

- Aggregate-specific persistence behavior
- External persistence abstraction
- Complex reusable query
- Testing boundary that genuinely matters

## 10.3 Unit of Work

EF Core DbContext already provides transaction/unit-of-work semantics.

Do not create another generic UnitOfWork abstraction unless a real requirement appears.

---

# 11. Data Model Direction

Initial core entities:

```text
User
Role
Permission
UserRole
RolePermission

Branch
Department

Customer
CustomerContact
CustomerNote
CustomerAttachment

Ticket
TicketCategory
TicketPriority
TicketAssignment
TicketMessage
TicketHistory
TicketAttachment

SlaPolicy
SlaTarget
SlaEscalationRule

KnowledgeArticle
KnowledgeCategory

Notification
AuditLog

Integration
ChannelConfiguration

AiSuggestion
AiConversation
```

The exact schema must be refined during implementation based on actual business workflows.

## 11.1 Multi-branch / multi-department

All business data that belongs to a branch/department should carry the appropriate ownership/context.

Prefer an explicit context model such as:

```text
Organization
 ├── Branch
 │    └── Department
 │         └── Users / Tickets / ...
```

Do not hard-code a single branch assumption.

Every feature must explicitly answer:

- What organization owns this?
- What branch does it belong to?
- What department owns it?
- Who can access it?

---

# 12. Soft Delete

Use soft delete only where business history should remain available.

Good candidates:

- Customers
- Knowledge articles
- Configuration records

Do not blindly soft-delete transactional/history records such as ticket history.

A soft-deleted record must not accidentally appear in normal queries.

Use EF Core global query filters carefully, especially for administrative/reporting queries.

---

# 13. Auditing

Audit important security and business actions:

```text
Actor
Action
EntityType
EntityId
Timestamp
CorrelationId
OldValues
NewValues
IpAddress
UserAgent
```

Never log:

- Passwords
- Access tokens
- Refresh tokens
- Secrets
- Full sensitive customer data unless explicitly required

---

# 14. API Layer

```text
CustomerSupportCrm.Api/
├── Endpoints/
├── Middleware/
│   ├── ExceptionHandlingMiddleware.cs
│   ├── CorrelationIdMiddleware.cs
│   └── RequestLoggingMiddleware.cs
│
├── Authentication/
├── Authorization/
├── OpenApi/
├── Localization/
├── Health/
├── Configuration/
└── Program.cs
```

Prefer endpoint definitions close to their feature slice where practical.

For example:

```text
Application/
└── Features/
    └── Tickets/
        └── Create/
            └── CreateTicketEndpoint.cs
```

The endpoint should be a thin HTTP adapter.

It must not contain business rules.

---

# 15. API Contract

All API responses must follow a predictable contract.

## 15.1 Success

```json
{
  "success": true,
  "data": {},
  "message": null,
  "errors": [],
  "meta": null,
  "correlationId": "..."
}
```

## 15.2 Error

```json
{
  "success": false,
  "data": null,
  "message": "An error occurred.",
  "errors": [
    {
      "code": "TICKET_NOT_FOUND",
      "message": "Ticket was not found.",
      "field": null
    }
  ],
  "meta": null,
  "correlationId": "..."
}
```

## 15.3 Validation error

```json
{
  "success": false,
  "data": null,
  "message": "Validation failed.",
  "errors": [
    {
      "code": "REQUIRED",
      "message": "Title is required.",
      "field": "title"
    }
  ],
  "meta": null,
  "correlationId": "..."
}
```

## 15.4 Pagination

```json
{
  "success": true,
  "data": [],
  "message": null,
  "errors": [],
  "meta": {
    "page": 1,
    "pageSize": 25,
    "totalCount": 250,
    "totalPages": 10
  },
  "correlationId": "..."
}
```

The exact response wrapper should be implemented once and reused consistently.

---

# 16. API Endpoint Conventions

Use resource-oriented routes.

Examples:

```text
GET    /api/v1/tickets
GET    /api/v1/tickets/{id}
POST   /api/v1/tickets
PUT    /api/v1/tickets/{id}
DELETE /api/v1/tickets/{id}

POST   /api/v1/tickets/{id}/assign
POST   /api/v1/tickets/{id}/status
POST   /api/v1/tickets/{id}/messages
POST   /api/v1/tickets/{id}/escalate
```

Avoid RPC-style routes unless the operation is genuinely an action.

---

# 17. API Versioning

Start with:

```text
/api/v1/...
```

Use URL versioning for public/stable APIs.

Do not create `v2` preemptively.

Introduce a new version only for a breaking contract change.

---

# 18. Validation

Use FluentValidation at the Application boundary.

Pipeline:

```text
HTTP Request
   ↓
Authentication
   ↓
Authorization
   ↓
MediatR Validation Behavior
   ↓
Command/Query Handler
   ↓
Domain Rules
```

Validation responsibilities:

### Application validation

- Required fields
- Format
- Input ranges
- Request-level rules
- Cross-field request rules

### Domain validation

- Business invariants
- State transitions
- Aggregate consistency

Do not duplicate domain rules in validators.

---

# 19. Error Handling

Use global exception handling.

Map known errors to stable error codes.

Example:

```text
VALIDATION_ERROR
UNAUTHORIZED
FORBIDDEN
NOT_FOUND
CONFLICT
BUSINESS_RULE_VIOLATION
RATE_LIMITED
INTERNAL_ERROR
```

Never expose stack traces in production.

Exception details go to logs, not API responses.

Localization should apply to user-facing messages.

Error codes remain stable and language-neutral.

---

# 20. Localization

Supported languages:

```text
ar
en
```

Backend:

```text
Resources/
├── Messages.ar.resx
└── Messages.en.resx
```

Use request language negotiation, e.g.:

```text
Accept-Language: ar
Accept-Language: en
```

The backend should return localized user-facing messages where appropriate.

Frontend also maintains localized UI resources.

Do not use translated text as a machine-readable error identifier.

Use:

```text
code = "TICKET_NOT_FOUND"
message = localized text
```

---

# 21. Authentication & Authorization

## 21.1 Authentication

Use:

```text
Access Token
+
Refresh Token
```

Access tokens should be short-lived.

Refresh tokens should:

- Be long-lived relative to access tokens
- Be rotated
- Be revocable
- Be stored securely
- Be associated with a user/session/device where appropriate
- Be stored hashed server-side where possible

## 21.2 Passwords

Use a modern password hashing implementation such as ASP.NET Core Identity's password hasher or an equivalent reviewed implementation.

Never:

- Encrypt passwords
- Store plaintext passwords
- Log passwords

## 21.3 Authorization

Use both:

```text
Role
+
Permission
```

Examples:

```text
tickets.view
tickets.create
tickets.update
tickets.assign
tickets.delete

customers.view
customers.create
customers.update

reports.view
users.manage
roles.manage
settings.manage
```

Use resource-level authorization where necessary.

Example:

```text
Agent can view tickets assigned to their department.
Manager can view department tickets.
Administrator can view all authorized branches.
```

Do not rely on UI hiding alone.

Authorization must be enforced server-side.

---

# 22. Current User Abstraction

Application code should use:

```text
ICurrentUser
```

with information such as:

```text
UserId
OrganizationId
BranchId
DepartmentIds
Roles
Permissions
Culture
```

The Application layer must not depend directly on `HttpContext.User`.

Infrastructure/API adapts HTTP claims to the abstraction.

---

# 23. Security Strategy

Implement:

- JWT authentication
- Refresh-token rotation
- Role and permission authorization
- Resource authorization
- Password hashing
- HTTPS
- Strict CORS
- Rate limiting
- Input validation
- Request-size limits
- Secure headers
- Secure cookies where cookies are used
- Secret management
- Audit logging
- Sensitive-data redaction
- Dependency scanning
- Container image scanning
- OWASP API Security controls

## 23.1 CSRF

CSRF protection is primarily required when browser authentication relies on cookies.

If the application uses an Authorization header with a bearer token and does not authenticate requests through ambient cookies, traditional cookie-based CSRF risk is reduced.

If refresh tokens are stored in HttpOnly cookies, implement appropriate CSRF protection for refresh/logout endpoints.

## 23.2 CORS

Allow only known frontend origins per environment.

Never use unrestricted production:

```text
AllowAnyOrigin
```

with credentialed requests.

## 23.3 Rate limiting

Apply stricter limits to:

- Login
- Refresh token
- Password reset
- Public customer portal
- AI endpoints
- Expensive searches
- File uploads

---

# 24. Secrets & Configuration

Use Options pattern:

```text
JwtOptions
DatabaseOptions
StorageOptions
EmailOptions
WhatsAppOptions
SmsOptions
AiOptions
RateLimitOptions
```

Configuration hierarchy:

```text
appsettings.json
appsettings.{Environment}.json
Environment Variables
Secret Store
```

Never commit production secrets.

Local development can use:

```text
dotnet user-secrets
```

Production should use a managed secret store.

---

# 25. Logging & Correlation

Use Serilog structured logging.

Every request gets:

```text
CorrelationId
RequestId
UserId
Route
Method
StatusCode
ElapsedMs
```

Correlation ID behavior:

1. Read incoming correlation ID if trusted/allowed.
2. Otherwise generate one.
3. Put it in logging context.
4. Return it in response headers.
5. Include it in API response body.

Example:

```text
X-Correlation-Id
```

Never log:

```text
Authorization
password
refreshToken
secret
```

---

# 26. Health Checks

Expose:

```text
/health/live
/health/ready
```

Liveness checks process health.

Readiness checks dependencies required to serve traffic, such as PostgreSQL.

Do not expose sensitive infrastructure details publicly.

---

# 27. Background Jobs

Use background processing for:

- SLA monitoring
- Escalations
- Notifications
- Email sending
- Integration retries
- AI processing
- Scheduled reports

Do not run long-running work inside the HTTP request.

A production implementation can use a durable job infrastructure when reliability requires it.

For simple in-process work, use hosted services only when losing the job on process restart is acceptable.

---

# 28. AI Architecture

AI must be an isolated capability.

```text
Application
 └── Ai/
     ├── SummarizeTicket
     ├── SuggestReply
     ├── CategorizeTicket
     ├── SuggestSolution
     └── Chatbot
```

Do not allow AI provider SDKs into Domain.

Use:

```text
IAiService
```

at the Application boundary and provider implementations in Infrastructure.

AI output must be treated as untrusted generated content.

Do not automatically execute high-impact actions based solely on AI output.

---

# 29. Communication Channels

The product requirements include:

```text
Email
WhatsApp
Live Chat
SMS
Web Forms
```

Model channels using a provider-neutral abstraction.

```text
ICommunicationChannel
IMessageSender
IMessageReceiver
```

Provider implementations belong in Infrastructure.

Do not create separate ticket business logic for every provider.

Normalize incoming messages into a common application model.

---

# 30. Customer Portal

The customer portal should reuse the same backend domain and API contracts.

Portal capabilities from the requirements:

- Submit tickets
- Track requests
- View history
- Access FAQs
- Submit feedback fileciteturn0file0L43-L48

Use separate authorization policies for customer users.

Never expose internal agent-only data.

---

# 31. Reporting

Reports should be implemented as query slices.

Examples:

```text
Features/
└── Reports/
    ├── TicketVolume/
    ├── SlaPerformance/
    ├── AgentPerformance/
    ├── CustomerSatisfaction/
    └── ManagementDashboard/
```

Reporting queries should use projections and database-side aggregation.

Do not load millions of records into application memory.

---

# 32. Frontend Repository

Repository:

```text
customer-support-crm-web/
```

Use modern Angular with:

- Standalone components
- Feature-based architecture
- Lazy loading
- Signals where useful
- Reactive forms
- Typed APIs
- Functional guards/interceptors where appropriate
- Dependency injection
- Strict TypeScript settings

---

# 33. Frontend Structure

```text
customer-support-crm-web/
├── src/
│   ├── app/
│   │   ├── core/
│   │   │   ├── auth/
│   │   │   ├── http/
│   │   │   ├── guards/
│   │   │   ├── interceptors/
│   │   │   ├── permissions/
│   │   │   ├── config/
│   │   │   ├── localization/
│   │   │   └── layout/
│   │   │
│   │   ├── shared/
│   │   │   ├── components/
│   │   │   ├── directives/
│   │   │   ├── pipes/
│   │   │   ├── forms/
│   │   │   ├── tables/
│   │   │   ├── dialogs/
│   │   │   ├── pagination/
│   │   │   ├── states/
│   │   │   └── utilities/
│   │   │
│   │   ├── features/
│   │   │   ├── auth/
│   │   │   ├── customers/
│   │   │   ├── tickets/
│   │   │   ├── dashboard/
│   │   │   ├── knowledge-base/
│   │   │   ├── sla/
│   │   │   ├── reports/
│   │   │   ├── users/
│   │   │   ├── roles/
│   │   │   ├── permissions/
│   │   │   ├── branches/
│   │   │   ├── departments/
│   │   │   ├── customer-portal/
│   │   │   └── settings/
│   │   │
│   │   ├── app.component.ts
│   │   └── app.routes.ts
│   │
│   ├── assets/
│   ├── environments/
│   ├── styles/
│   └── index.html
│
├── public/
├── angular.json
├── package.json
├── tsconfig.json
└── README.md
```

---

# 34. Angular Core

`core/` contains singleton application infrastructure.

```text
core/
├── auth/
├── http/
├── guards/
├── interceptors/
├── permissions/
├── config/
├── localization/
└── layout/
```

Do not put feature-specific business logic here.

---

# 35. Angular Shared

`shared/` contains genuinely reusable UI and utilities.

Examples:

```text
shared/
├── components/
│   ├── app-button/
│   ├── empty-state/
│   ├── loading-state/
│   ├── error-state/
│   ├── confirm-dialog/
│   └── page-header/
│
├── tables/
│   ├── data-table/
│   ├── table-actions/
│   └── column-definition/
│
├── pagination/
├── filters/
├── forms/
├── dialogs/
├── directives/
└── pipes/
```

A component belongs in `shared` only when at least two independent features genuinely reuse it or it is clearly a platform-level UI primitive.

---

# 36. Angular Feature Slice

Example:

```text
features/
└── tickets/
    ├── data-access/
    │   ├── ticket-api.service.ts
    │   ├── ticket.models.ts
    │   └── ticket.mappers.ts
    │
    ├── pages/
    │   ├── ticket-list/
    │   ├── ticket-details/
    │   └── create-ticket/
    │
    ├── components/
    │   ├── ticket-status-badge/
    │   ├── ticket-priority-badge/
    │   ├── ticket-timeline/
    │   └── ticket-assignment/
    │
    ├── state/
    └── tickets.routes.ts
```

Do not create a global `TicketService` if the service is feature-specific.

---

# 37. Angular State Management

Do not introduce a global state library by default.

Use:

1. Component state/signals for local state.
2. Feature-level state when multiple components need shared state.
3. A dedicated global state solution only for truly global cross-feature state.

Good global candidates:

```text
Authenticated user
Permissions
Application settings
Current language
```

Avoid storing every API response globally.

---

# 38. Generic API Client

Create a typed API layer:

```text
ApiClient
ApiResponse<T>
ApiError
PaginationMeta
PagedResponse<T>
```

Feature API services use it:

```text
TicketApiService
CustomerApiService
KnowledgeBaseApiService
```

Do not let every component manually construct URLs and headers.

---

# 39. HTTP Interceptors

Implement functional interceptors for:

```text
auth.interceptor
refresh-token.interceptor
correlation-id.interceptor
error.interceptor
localization.interceptor
```

Responsibilities:

### Auth

Adds access token.

### Refresh

Handles expired access token according to one centralized refresh flow.

Prevent concurrent requests from triggering multiple refresh operations.

### Correlation

Adds:

```text
X-Correlation-Id
```

### Localization

Adds:

```text
Accept-Language
```

### Error

Converts API errors into a consistent frontend error model.

Do not show raw backend exception text to users.

---

# 40. Authentication Storage

Prefer a design that minimizes XSS exposure.

Recommended approach:

- Short-lived access token held in application memory where practical.
- Refresh token in a Secure, HttpOnly, SameSite cookie when backend deployment supports it.
- Never store refresh tokens in `localStorage`.

If the architecture uses cookie-based refresh, protect the refresh endpoint against CSRF.

---

# 41. Angular Authorization

Centralize permission checking.

Example:

```text
PermissionService.has("tickets.assign")
```

Use guards for route-level access.

Use directives/components for UI visibility.

But remember:

```text
Frontend authorization = UX
Backend authorization = security
```

Never rely on Angular permissions to secure data.

---

# 42. Localization and RTL/LTR

Supported:

```text
Arabic
English
```

The application must dynamically support:

```text
dir="rtl"
dir="ltr"
```

Centralize:

```text
LanguageService
DirectionService
TranslationService
```

Changing language should update:

- UI translations
- Direction
- Date formatting
- Number formatting
- API `Accept-Language`

Avoid hardcoded user-facing strings inside feature components.

---

# 43. Responsive UI

Build mobile-friendly layouts from the beginning.

Required considerations:

- Responsive navigation
- Mobile ticket details
- Responsive tables
- Alternative card layout where tables do not fit
- Touch-friendly controls
- Responsive dialogs
- Keyboard accessibility

The source requirements explicitly call for web and mobile-friendly behavior. fileciteturn0file0L65-L70

---

# 44. Accessibility

Follow WCAG-oriented practices:

- Semantic HTML
- Keyboard navigation
- Visible focus states
- Proper labels
- ARIA only when needed
- Sufficient contrast
- Screen-reader-friendly status messages
- Dialog focus management
- Error association with form fields

Do not make accessibility the responsibility of each feature alone; shared components should establish accessible defaults.

---

# 45. Reusable Frontend Components

Build these as shared primitives:

```text
DataTable
Pagination
FilterBar
SearchBox
SortHeader
LoadingState
EmptyState
ErrorState
ConfirmDialog
Modal
Drawer
FormField
Select
DatePicker
FileUpload
StatusBadge
PriorityBadge
UserAvatar
PageHeader
Breadcrumbs
Toast/Notification
```

Feature-specific components should remain in their feature.

---

# 46. Generic Tables

A generic table should support:

```text
Columns
Sorting
Pagination
Loading
Empty state
Row actions
Selection
Responsive behavior
```

Do not make the table component responsible for business logic.

---

# 47. Generic Forms

Use reusable form controls and validation presentation.

Do not create one universal "form builder" too early.

Prefer typed feature forms composed from shared controls.

---

# 48. Generic Dialogs

Use shared dialog infrastructure for:

- Confirm
- Form modal
- Details modal
- Delete confirmation

Feature-specific dialog content remains feature-specific.

---

# 49. Frontend API Contract Models

Recommended:

```text
core/http/models/
├── api-response.ts
├── api-error.ts
├── pagination.ts
└── api-result.ts
```

Example TypeScript model:

```text
ApiResponse<T>
PagedResponse<T>
ApiError
ValidationError
PaginationMeta
```

These models must mirror the backend contract exactly.

---

# 50. End-to-End Feature Example: Create Ticket

This section is the reference implementation pattern for all future features.

## 50.1 User flow

```text
Angular Ticket Form
       ↓
Ticket API Service
       ↓
POST /api/v1/tickets
       ↓
API Endpoint
       ↓
CreateTicketCommand
       ↓
Validation Behavior
       ↓
CreateTicketHandler
       ↓
Ticket Aggregate
       ↓
EF Core / PostgreSQL
       ↓
Domain Event
       ↓
Notification / Audit / Integration
       ↓
Standard API Response
       ↓
Angular ApiClient
       ↓
Feature UI
```

## 50.2 Backend files

```text
Application/
└── Features/
    └── Tickets/
        └── Create/
            ├── CreateTicketCommand.cs
            ├── CreateTicketValidator.cs
            ├── CreateTicketHandler.cs
            ├── CreateTicketResponse.cs
            └── CreateTicketEndpoint.cs
```

## 50.3 Domain behavior

The handler should not directly manipulate ticket fields.

Conceptually:

```text
Ticket.Create(...)
Ticket.AssignTo(...)
Ticket.SetPriority(...)
```

The aggregate enforces its own rules.

## 50.4 Frontend files

```text
features/
└── tickets/
    ├── data-access/
    │   └── ticket-api.service.ts
    │
    └── pages/
        └── create-ticket/
            ├── create-ticket.component.ts
            ├── create-ticket.component.html
            └── create-ticket.component.scss
```

The component owns presentation and form interaction.

The API service owns HTTP communication.

---

# 51. Ticket Domain Design

Tickets are a core aggregate.

A ticket should encapsulate:

```text
TicketNumber
Customer
Category
Priority
Status
AssignedAgent
Department
Branch
Sla
Messages
History
CreatedAt
UpdatedAt
```

Not every related object must be loaded into the aggregate for every query.

Commands operate through the aggregate.

Queries use optimized projections.

---

# 52. Ticket Status

Define explicit domain transitions.

Example:

```text
New
Open
PendingCustomer
PendingInternal
Resolved
Closed
Escalated
```

Do not allow arbitrary string status changes.

The domain determines valid transitions.

For example:

```text
Closed → Open
```

may require a dedicated reopen action rather than an unrestricted update.

---

# 53. SLA Architecture

SLA should be a domain capability, not a controller timer.

Model:

```text
SlaPolicy
SlaTarget
SlaEscalationRule
```

Track:

```text
FirstResponseDueAt
ResolutionDueAt
FirstRespondedAt
ResolvedAt
```

Background processing evaluates SLA state.

Emit business facts:

```text
SlaApproaching
SlaBreached
```

Notification handlers react to those events.

---

# 54. Knowledge Base

Knowledge Base requirements include:

- FAQs
- Help articles
- Solutions and guides
- Search fileciteturn0file0L31-L35

Suggested slices:

```text
KnowledgeBase/
├── Articles/
│   ├── Create
│   ├── Update
│   ├── Publish
│   ├── Archive
│   └── GetById
├── Categories/
└── Search/
```

Publishing should be a domain action, not a generic status setter.

---

# 55. Agent Dashboard

Dashboard queries should aggregate only the data needed by the UI.

Examples:

```text
MyTickets
MyTasks
SlaAtRisk
RecentCustomers
UnreadMessages
PendingEscalations
QuickReplies
```

Do not make the dashboard load complete ticket aggregates.

---

# 56. Notifications

Notification channels:

```text
In-App
Email
SMS
WhatsApp
```

Use a notification abstraction.

Notification preferences should be configurable where appropriate.

Do not make business handlers call external providers directly.

Prefer:

```text
Domain/Application event
        ↓
Notification handler
        ↓
Notification service
        ↓
Provider adapter
```

---

# 57. Attachments

Attachments must be stored outside PostgreSQL for actual file content.

Database stores metadata:

```text
Id
FileName
ContentType
Size
StorageKey
EntityType
EntityId
UploadedBy
CreatedAt
```

Use object storage for binary data.

Validate:

- File size
- MIME type
- Extension
- File signature where security requires it
- Malware scanning in production where appropriate

Never trust the client-provided MIME type alone.

---

# 58. Search

Start with PostgreSQL search capabilities.

Do not introduce Elasticsearch/OpenSearch on day one unless scale or search requirements justify it.

Abstract the search capability so an external search engine can be introduced later without changing feature contracts.

---

# 59. Testing Strategy

## 59.1 Domain tests

Test:

- Invariants
- State transitions
- Value objects
- Domain events
- Business rules

These should be fast and require no database.

## 59.2 Application tests

Test:

- Command behavior
- Query behavior
- Authorization
- Validation
- Application orchestration

Mock only true external boundaries.

Do not mock your own domain entities.

## 59.3 Integration tests

Use real PostgreSQL in a disposable test environment.

Test:

- EF mappings
- Migrations
- Queries
- Transactions
- Authorization integration
- API contracts

Avoid replacing the database with an in-memory fake for persistence correctness.

## 59.4 API tests

Test:

- HTTP status codes
- Request validation
- Authentication
- Authorization
- Response contract
- Error contract
- Correlation ID
- Localization

## 59.5 Frontend tests

Test:

- Components
- Signals/state
- Services
- Guards
- Interceptors
- Form validation
- Permission visibility

Mock HTTP at the network boundary.

Do not mock simple TypeScript functions unnecessarily.

## 59.6 E2E

Use an E2E framework such as Playwright.

Critical journeys:

```text
Login
Create customer
Create ticket
Assign ticket
Change status
Send ticket message
Resolve ticket
Customer portal submission
Permission denial
Arabic/English switching
```

---

# 60. What Not to Mock

Avoid mocking:

- Domain entities
- Value objects
- Pure functions
- Simple mappers
- EF Core internals

Mock/stub:

- Email providers
- WhatsApp providers
- SMS providers
- AI providers
- Object storage
- External ERP APIs
- External authentication providers

Use real PostgreSQL for integration tests.

---

# 61. CI/CD

Pipeline stages:

```text
Restore
↓
Build
↓
Format Check
↓
Static Analysis
↓
Unit Tests
↓
Integration Tests
↓
API Tests
↓
Security Scan
↓
Build Docker Image
↓
Container Scan
↓
Publish Artifact
↓
Deploy
↓
Smoke Test
```

Frontend pipeline:

```text
Install
↓
Lint
↓
Type Check
↓
Unit Tests
↓
Build
↓
E2E
↓
Security Scan
↓
Publish
```

---

# 62. Environments

Use:

```text
Development
Test
Staging
Production
```

Configuration must be environment-specific.

Do not build environment-specific business logic.

Use the same artifact promoted across environments where practical.

---

# 63. Docker

Backend Dockerfile should use multi-stage builds:

```text
SDK image
   ↓
restore
   ↓
build
   ↓
publish
   ↓
runtime image
```

Frontend should also use multi-stage builds and serve production assets through an appropriate web server/container.

Do not ship SDK tooling in production images.

---

# 64. Database Migration Strategy

Migrations are source-controlled.

Rules:

1. Developers create migrations.
2. Migration names are meaningful.
3. CI verifies migrations.
4. Production migrations are applied through a controlled deployment process.
5. Destructive migrations require explicit review.
6. Large data migrations should be separated from schema changes.
7. Never manually change production schema outside the migration/change-management process unless emergency procedures require it.

---

# 65. Code Quality

Use:

- Nullable reference types
- Treat warnings seriously
- `.editorconfig`
- `dotnet format`
- Static analyzers
- Dependency vulnerability scanning
- TypeScript strict mode
- ESLint
- Prettier where adopted by the frontend team

Prefer simple code over clever abstractions.

---

# 66. Modern C# Standards

Prefer current stable C# features appropriate to the selected .NET version:

- File-scoped namespaces
- Primary constructors where they improve clarity
- Records for immutable DTOs
- `required` members where useful
- Collection expressions
- Pattern matching
- Nullable reference types
- Async/await
- CancellationToken propagation
- `DateTimeOffset`/UTC for timestamps
- Strong typing for domain concepts

Do not use a modern language feature merely because it exists.

Readability wins.

---

# 67. DTO Standards

DTOs are boundary contracts.

Rules:

- Do not expose EF entities.
- Do not expose domain entities directly.
- Prefer immutable request/response models.
- Use explicit API response DTOs.
- Keep query responses optimized for their screen/use case.
- Do not create giant universal DTOs.

Naming:

```text
CreateTicketRequest
CreateTicketResponse
GetTicketByIdResponse
SearchTicketsResponse
```

---

# 68. Naming Conventions

Backend:

```text
PascalCase
```

Examples:

```text
CreateTicketCommand
CreateTicketHandler
TicketCreatedDomainEvent
```

Frontend:

```text
kebab-case file names
```

Examples:

```text
create-ticket.component.ts
ticket-api.service.ts
permission.service.ts
```

Classes/interfaces/types use PascalCase.

Methods and properties use camelCase.

---

# 69. Git Strategy

Use:

```text
main
develop
feature/*
bugfix/*
hotfix/*
release/*
```

If the team prefers trunk-based development, use short-lived branches and protect `main`.

For this project, default to:

```text
main
develop
feature/<ticket-id>-<short-name>
bugfix/<ticket-id>-<short-name>
hotfix/<ticket-id>-<short-name>
```

Protect `main` and `develop`.

Require:

- Pull request
- Successful CI
- Code review
- No unresolved comments

---

# 70. Commit Convention

Use Conventional Commits:

```text
feat: add ticket creation
fix: prevent invalid ticket transition
refactor: simplify ticket query
test: add ticket authorization tests
docs: update architecture plan
chore: update dependencies
```

Keep commits focused.

---

# 71. Code Review Guidelines

Review for:

1. Correct business behavior
2. Domain invariants
3. Authorization
4. Validation
5. Error handling
6. Performance
7. Security
8. Localization
9. Test coverage
10. Maintainability
11. Observability
12. Unnecessary abstraction

Do not approve code merely because tests pass.

---

# 72. Reusability Rules

## Generic

Good candidates:

- API response wrappers
- Pagination metadata
- Error model
- Authentication infrastructure
- Permission checking
- HTTP infrastructure
- Logging
- Localization
- Shared UI primitives

## Shared

Good candidates:

- Table
- Pagination
- Dialog
- Form controls
- Loading/empty/error states
- Date/number formatting
- Layout

## Feature-specific

Keep:

- Ticket assignment UI
- Ticket timeline
- Customer interaction timeline
- SLA configuration UI
- AI suggestion UI

inside their feature unless multiple features genuinely share the same concept.

## Infrastructure-specific

Keep:

- EF Core
- PostgreSQL
- Email provider SDKs
- WhatsApp provider SDKs
- Object storage SDKs
- AI provider SDKs

inside Infrastructure.

---

# 73. Anti-Patterns to Avoid

Do not introduce:

```text
GenericRepository<T>
GenericService<T>
GenericController<T>
BaseCrudService<T>
BaseCrudController<T>
```

unless a demonstrated requirement justifies them.

Avoid:

- Anemic domain models
- Fat controllers
- Business logic in Angular components
- Business logic in EF configurations
- Direct provider calls from domain
- Global state for every API response
- Giant shared modules
- Giant utility classes
- String-based permissions scattered through templates
- Returning entities directly from APIs
- Catching exceptions in every handler
- Logging secrets
- Duplicate validation rules everywhere

---

# 74. Observability

Production observability should cover:

```text
Logs
Metrics
Traces
Health
```

Minimum useful metrics:

- Request count
- Error count
- Request duration
- Database latency
- Authentication failures
- Rate-limit events
- Queue/job failures
- SLA breaches
- External provider failures

Correlation IDs must connect API requests with downstream logs.

---

# 75. Performance Guidelines

Backend:

- Use async I/O.
- Use cancellation tokens.
- Project query results.
- Avoid N+1 queries.
- Use appropriate indexes.
- Paginate large collections.
- Use database-side filtering/sorting.
- Cache only measured hot paths.
- Use background jobs for expensive work.

Frontend:

- Lazy-load features.
- Avoid unnecessary subscriptions.
- Prefer signals/local state where useful.
- Track list rendering correctly.
- Paginate large datasets.
- Debounce search inputs.
- Avoid loading full datasets for dropdowns/tables.

---

# 76. API Pagination / Filtering / Sorting

Standard query model:

```text
page
pageSize
search
sortBy
sortDirection
filters
```

Example:

```text
GET /api/v1/tickets?page=1&pageSize=25&search=refund&sortBy=createdAt&sortDirection=desc
```

Keep filter semantics consistent across resources.

Avoid building one overly generic dynamic query framework that becomes difficult to secure and maintain.

Feature-specific filters should remain feature-specific.

---

# 77. Notifications & User Messages

Distinguish:

```text
Technical API error
Business error
User notification
```

Examples:

```text
API error:
TICKET_NOT_FOUND

User notification:
Ticket assigned successfully.
```

The frontend should not infer messages from HTTP status alone.

Use stable codes and localized messages.

---

# 78. Documentation

Maintain:

```text
README.md
docs/architecture.md
docs/api-contract.md
docs/security.md
docs/development.md
```

Swagger/OpenAPI is the machine-readable API contract.

Every important architectural decision should be recorded as an ADR when it materially affects future development.

Suggested:

```text
docs/adr/
├── 0001-vertical-slice-architecture.md
├── 0002-authentication-strategy.md
├── 0003-api-contract.md
└── 0004-multi-branch-context.md
```

---

# 79. Development Workflow

For every feature:

```text
1. Understand business requirement
2. Identify aggregate/domain rules
3. Define API contract
4. Define command/query
5. Implement validation
6. Implement domain behavior
7. Implement persistence
8. Implement endpoint
9. Add tests
10. Implement Angular data-access
11. Implement UI
12. Add authorization
13. Add localization
14. Add telemetry
15. Run CI
16. Code review
17. Merge
```

---

# 80. Feature Definition of Done

A feature is not complete until:

- Domain rules are implemented.
- Authorization is enforced.
- Validation exists.
- API contract is documented.
- Localization exists.
- Error behavior is consistent.
- Logging/correlation exists.
- Unit tests exist.
- Integration tests exist where persistence is involved.
- Frontend tests exist where applicable.
- Responsive UI is verified.
- Accessibility basics are verified.
- No secrets are introduced.
- CI passes.
- Documentation is updated where needed.

---

# 81. Implementation Phases

## Phase 0 — Repository Foundation

- Create backend repository.
- Create frontend repository.
- Create solution/projects.
- Configure .NET version.
- Configure Angular version.
- Configure formatting/analyzers.
- Configure CI skeleton.
- Configure Docker.
- Create README.
- Establish branch protections.

## Phase 1 — Backend Platform

- PostgreSQL
- EF Core
- Configuration
- Global exception handling
- API response contract
- Correlation ID
- Serilog
- Health checks
- Swagger/OpenAPI
- Localization
- Validation pipeline

## Phase 2 — Identity & Authorization

- Users
- Roles
- Permissions
- JWT
- Refresh tokens
- Current-user abstraction
- Authorization policies
- Audit logging

## Phase 3 — Organization Context

- Organization
- Branches
- Departments
- Context-aware authorization

## Phase 4 — Customers

- Customer CRUD
- Contacts
- Notes
- Attachments
- Interaction history

## Phase 5 — Tickets

- Create
- List/search
- Get details
- Update
- Assign
- Status transitions
- Priority
- Categories
- Messages
- History
- Attachments

## Phase 6 — Agent Workspace

- Assigned tickets
- Customer information
- Tasks
- Reminders
- Quick replies
- Collaboration

## Phase 7 — SLA & Automation

- SLA policies
- Response targets
- Resolution targets
- Assignment rules
- Escalation rules
- Alerts

## Phase 8 — Knowledge Base

- Categories
- FAQs
- Articles
- Solutions/guides
- Search
- Publishing

## Phase 9 — Customer Portal

- Customer authentication
- Ticket submission
- Tracking
- History
- FAQ
- Feedback

## Phase 10 — Communications

- Email
- WhatsApp
- Live chat
- SMS
- Web forms

Implement providers through adapters.

## Phase 11 — Reports

- Ticket reports
- SLA performance
- Agent performance
- Customer satisfaction
- Management dashboards

The reporting scope follows the supplied requirements. fileciteturn0file0L49-L54

## Phase 12 — AI

- Ticket summaries
- Suggested replies
- Automatic categorization
- Suggested solutions
- AI chatbot

These capabilities are explicitly listed in the supplied requirements. fileciteturn0file0L37-L42

## Phase 13 — Advanced Integrations

- Public APIs
- ERP
- External systems
- Provider integrations

The supplied requirements explicitly include APIs, ERP, Email/SMS/WhatsApp and external systems. fileciteturn0file0L60-L64

---

# 82. Initial Folder Tree — Backend

```text
customer-support-crm-api/
├── src/
│   ├── CustomerSupportCrm.Api/
│   │   ├── Middleware/
│   │   ├── OpenApi/
│   │   ├── Health/
│   │   └── Program.cs
│   │
│   ├── CustomerSupportCrm.Application/
│   │   ├── Abstractions/
│   │   ├── Behaviors/
│   │   ├── Features/
│   │   │   ├── Authentication/
│   │   │   ├── Customers/
│   │   │   ├── Tickets/
│   │   │   ├── Dashboard/
│   │   │   ├── Sla/
│   │   │   ├── KnowledgeBase/
│   │   │   ├── Notifications/
│   │   │   ├── Reports/
│   │   │   ├── Users/
│   │   │   ├── Roles/
│   │   │   ├── Permissions/
│   │   │   ├── Branches/
│   │   │   ├── Departments/
│   │   │   ├── CustomerPortal/
│   │   │   ├── Integrations/
│   │   │   └── Ai/
│   │   └── DependencyInjection.cs
│   │
│   ├── CustomerSupportCrm.Domain/
│   │   ├── Common/
│   │   ├── Customers/
│   │   ├── Tickets/
│   │   ├── Sla/
│   │   ├── KnowledgeBase/
│   │   ├── Users/
│   │   ├── Organizations/
│   │   ├── Departments/
│   │   ├── Notifications/
│   │   └── Audit/
│   │
│   ├── CustomerSupportCrm.Infrastructure/
│   │   ├── Persistence/
│   │   ├── Identity/
│   │   ├── Authentication/
│   │   ├── Authorization/
│   │   ├── Email/
│   │   ├── Sms/
│   │   ├── WhatsApp/
│   │   ├── Files/
│   │   ├── Ai/
│   │   ├── Integrations/
│   │   ├── BackgroundJobs/
│   │   └── DependencyInjection.cs
│   │
│   └── CustomerSupportCrm.Contracts/
│       ├── Common/
│       ├── Authentication/
│       ├── Customers/
│       ├── Tickets/
│       └── Reports/
│
├── tests/
│   ├── CustomerSupportCrm.Domain.Tests/
│   ├── CustomerSupportCrm.Application.Tests/
│   ├── CustomerSupportCrm.IntegrationTests/
│   └── CustomerSupportCrm.Api.Tests/
│
└── docs/
    ├── architecture.md
    ├── api-contract.md
    ├── security.md
    ├── development.md
    └── adr/
```

---

# 83. Initial Folder Tree — Frontend

```text
customer-support-crm-web/
├── src/
│   ├── app/
│   │   ├── core/
│   │   │   ├── auth/
│   │   │   ├── guards/
│   │   │   ├── interceptors/
│   │   │   ├── permissions/
│   │   │   ├── http/
│   │   │   ├── config/
│   │   │   ├── localization/
│   │   │   └── layout/
│   │   │
│   │   ├── shared/
│   │   │   ├── components/
│   │   │   ├── forms/
│   │   │   ├── tables/
│   │   │   ├── dialogs/
│   │   │   ├── pagination/
│   │   │   ├── filters/
│   │   │   ├── states/
│   │   │   ├── directives/
│   │   │   └── pipes/
│   │   │
│   │   ├── features/
│   │   │   ├── auth/
│   │   │   ├── customers/
│   │   │   ├── tickets/
│   │   │   ├── dashboard/
│   │   │   ├── knowledge-base/
│   │   │   ├── sla/
│   │   │   ├── reports/
│   │   │   ├── users/
│   │   │   ├── roles/
│   │   │   ├── permissions/
│   │   │   ├── branches/
│   │   │   ├── departments/
│   │   │   ├── customer-portal/
│   │   │   └── settings/
│   │   ├── app.component.ts
│   │   └── app.routes.ts
│   │
│   ├── assets/
│   ├── environments/
│   └── styles/
│
├── e2e/
├── public/
├── angular.json
├── package.json
├── tsconfig.json
└── README.md
```

---

# 84. AI Coding Agent Rules

This document is the source of truth for implementation.

An AI coding agent must:

1. Read this file before creating application code.
2. Follow the repository boundaries.
3. Implement features vertically.
4. Avoid creating generic abstractions without a demonstrated need.
5. Keep domain rules inside domain models.
6. Keep infrastructure implementations out of Domain.
7. Keep API endpoints thin.
8. Use commands for writes and queries for reads.
9. Add tests with every feature.
10. Add localization for every user-facing message.
11. Add authorization to every protected feature.
12. Preserve the API response contract.
13. Propagate cancellation tokens.
14. Preserve correlation IDs.
15. Never commit secrets.
16. Never expose domain entities as API responses.
17. Never bypass authorization because the UI hides a feature.
18. Never introduce a new library when the platform already provides a suitable capability without a clear reason.
19. Update documentation when an architectural decision changes.
20. Prefer the simplest production-ready implementation.

---

# 85. Implementation Acceptance Checklist

Before declaring the architecture implemented:

## Backend

- [ ] .NET latest stable selected and pinned
- [ ] PostgreSQL configured
- [ ] EF Core configured
- [ ] Domain project isolated
- [ ] Vertical Slices implemented
- [ ] CQRS implemented
- [ ] MediatR pipeline configured
- [ ] Validation pipeline configured
- [ ] API contract implemented
- [ ] Global exception handling implemented
- [ ] JWT authentication implemented
- [ ] Refresh tokens implemented
- [ ] Roles/permissions implemented
- [ ] Current user abstraction implemented
- [ ] Localization implemented
- [ ] Serilog configured
- [ ] Correlation ID implemented
- [ ] Health checks implemented
- [ ] Swagger JWT configured
- [ ] Options pattern implemented
- [ ] Auditing implemented where required
- [ ] Soft delete implemented where required
- [ ] Pagination/filtering/sorting patterns implemented
- [ ] Integration tests use PostgreSQL
- [ ] Docker image created
- [ ] CI/CD pipeline created
- [ ] Security scanning enabled

## Frontend

- [ ] Angular latest stable selected and pinned
- [ ] Standalone components
- [ ] Feature-based structure
- [ ] Lazy-loaded routes
- [ ] Shared UI primitives
- [ ] Typed API client
- [ ] HTTP interceptors
- [ ] JWT flow
- [ ] Refresh flow
- [ ] Correlation ID
- [ ] Localization
- [ ] RTL/LTR
- [ ] Permission service
- [ ] Guards
- [ ] Reusable tables
- [ ] Reusable forms
- [ ] Reusable dialogs
- [ ] Loading/empty/error states
- [ ] Responsive UI
- [ ] Accessibility baseline
- [ ] Unit tests
- [ ] E2E tests
- [ ] CI pipeline

---

# 86. Final Architectural Decision Summary

The project should be implemented as **two independently deployable repositories**:

```text
customer-support-crm-api
customer-support-crm-web
```

The backend uses:

```text
ASP.NET Core
+
Vertical Slice Architecture
+
DDD
+
Clean Architecture dependency boundaries
+
CQRS
+
MediatR
+
EF Core
+
PostgreSQL
```

The frontend uses:

```text
Angular
+
Standalone Components
+
Feature-based architecture
+
Lazy loading
+
Typed API access
+
Signals/local state where useful
+
Reusable UI primitives
```

Both applications share a deliberately small set of contracts and conventions rather than sharing implementation details.

The primary architectural rule is:

> **Organize code around business capabilities and use cases, not around technical layers.**

The primary domain rule is:

> **Business invariants belong to the domain model.**

The primary reuse rule is:

> **Abstract only when the abstraction represents a real reusable concept.**

The primary security rule is:

> **The backend is the security boundary; frontend authorization is not a security mechanism.**

The primary delivery rule is:

> **Every feature is implemented end-to-end with domain behavior, API contract, authorization, localization, observability, tests and UI.**

This blueprint is intentionally implementation-oriented so an AI coding agent can use it as the starting source of truth without first restructuring a conventional layered codebase.
