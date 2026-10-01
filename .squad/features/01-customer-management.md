# 01 — Customer Management

> **Source:** AZM Squad Customer Support CRM — Core Features §1
> **Implementation phase:** Phase 4 — Customers
> **Status:** Done (backend + frontend) — backend plans [23–24, 46](../plans/customer-management/00-overview.md), frontend plan [10](../plans/frontend/10-story-customers-ui.md)
> **Build priority:** 4 (after Platform, Identity, Organization Context)

## Summary

Keep a single, complete view of every customer: who they are, how to reach them, what has happened with them, and any supporting notes or files.

## Scope

| Sub-feature | Description |
|-------------|-------------|
| Customer profiles | Create, view, update, soft-delete a customer (individual or company). |
| Contact details | Multiple contacts per customer (phone, email, WhatsApp, address), with a primary contact. |
| Interaction history | Timeline of every touchpoint: tickets, messages, calls, portal activity, notes. |
| Notes and attachments | Internal notes and files attached to the customer record. |

## User stories

- As an **agent**, I can search customers by name, phone, email or customer number so I can find the right record quickly.
- As an **agent**, I can create and edit a customer profile with multiple contact methods.
- As an **agent**, I can see a customer's full interaction history on one timeline.
- As an **agent**, I can add internal notes and upload attachments to a customer.
- As a **supervisor**, I can restrict customers to the branch/department that owns them.

## Acceptance criteria

- [ ] Customer CRUD with soft delete and audit trail.
- [ ] Duplicate detection on phone/email within the organization.
- [ ] Contacts are validated per type (E.164 phone, valid email).
- [ ] Interaction history is a paged, read-optimized projection (not full aggregate loads).
- [ ] Attachments stored in object storage; DB stores metadata only; size/MIME/extension/signature validated.
- [ ] Notes are internal-only and never exposed through the customer portal.
- [ ] Every record carries Organization / Branch / Department ownership and access is enforced.
- [ ] Arabic/English labels and RTL layout.

## Domain model

```text
Customer (aggregate)
 ├── CustomerContact
 ├── CustomerNote
 └── CustomerAttachment
```

Value objects: `CustomerNumber`, `PhoneNumber`, `EmailAddress`, `Address`.

## Backend slices

```text
Features/Customers/
├── CreateCustomer
├── UpdateCustomer
├── DeleteCustomer
├── GetCustomerById
├── ListCustomers (search, filter, paging)
├── Contacts/ (Add, Update, Remove, SetPrimary)
├── Notes/ (Add, Update, Delete, List)
├── Attachments/ (Upload, Download, Delete, List)
└── GetInteractionHistory
```

## Frontend

- `features/customers`: list page, details page (tabs: Profile, Contacts, History, Notes, Attachments), create/edit form.

## Dependencies

- Platform (12), Security & Administration (10), Organization context.
- Feeds: Ticket Management (02), Agent Dashboard (04), Customer Portal (08).

## Permissions

`Customers.View`, `Customers.Create`, `Customers.Update`, `Customers.Delete`, `Customers.Notes.Manage`, `Customers.Attachments.Manage`
