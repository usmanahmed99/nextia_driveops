# DrivingOps â€” Feature Documentation

This folder documents **every feature that exists in the DrivingOps concept today** and what is required to turn each one into a real production implementation.

The concept lives across four repos:

- [`dops-frontend`](../dops-frontend) â€” React + Vite SPA. Fully built screens, currently driven by either an in-memory mock adapter or the HTTP adapter pointing at `dops-api`.
- [`dops-api`](../dops-api) â€” FastAPI modular monolith. Real endpoints, SQLAlchemy models, Alembic, seed data; no real integrations (WhatsApp/email/payments/blob/identity provider) and several core algorithms are placeholders.
- [`dops-automation-worker`](../dops-automation-worker) â€” empty repo. Place-holder for the Service Bus consumer that will execute automation flows.
- [`dops-infra`](../dops-infra) â€” Bicep + Azure DevOps pipeline. Provisions a working dev environment (Container Apps, PostgreSQL, ACR, Key Vault, Storage, Service Bus, App Config, Log Analytics, App Insights).

## How To Read These Docs

Each feature doc follows the same structure:

1. **Purpose** â€” what the feature is for.
2. **Current state** â€” what is actually implemented today across frontend, API, worker, infra.
3. **Data model** â€” current tables/schemas.
4. **API surface** â€” endpoints today.
5. **UX surfaces** â€” pages/components today.
6. **Gaps for real implementation** â€” what is missing before this feature is production-grade.
7. **Acceptance criteria** â€” definition of done for the real implementation.
8. **Dependencies** â€” other docs/features this one depends on.

> "Real implementation" in this folder means: real persistence, real integrations, real authorization, real validation, real observability. It does **not** introduce new features beyond what the concept already shows.

## Feature Index

| #  | Document | Concept status | Real-impl size |
| -- | -------- | -------------- | -------------- |
| 00 | [Product Overview & Personas](./00-overview.md) | n/a | n/a |
| 01 | [Cross-Repo Architecture](./01-architecture.md) | dev infra running | M |
| 02 | [Multi-Tenancy, Roles, Permissions](./02-multitenancy-rbac.md) | demo-only | L |
| 03 | [Authentication & Sessions](./03-authentication.md) | demo login | L |
| 04 | [Learners](./04-learners.md) | CRUD + workspace stub | M |
| 05 | [Instructors](./05-instructors.md) | CRUD + availability slots | M |
| 06 | [Scheduling & Sessions](./06-scheduling.md) | sessions CRUD; **generation = placeholder** | **XL** |
| 07 | [Typical School Schedule (Templates)](./07-school-schedule-templates.md) | **NOT IMPLEMENTED** | **XL** |
| 08 | [Automations & Communications](./08-automations.md) | flows CRUD; no transport; worker empty | XL |
| 09 | [Invoices & Payments](./09-invoices-payments.md) | invoice/payment CRUD; no PSP | L |
| 10 | [Certificates](./10-certificates.md) | approve/void; no PDF/storage | M |
| 11 | [Documents](./11-documents.md) | upload links placeholder | M |
| 12 | [Public Site Builder](./12-public-site.md) | pages + theme; no live render | L |
| 13 | [Learner & Instructor Portals](./13-portals.md) | mobile UI present | M |
| 14 | [Reports & Dashboard](./14-reports-dashboard.md) | charts on mock data | M |
| 15 | [Settings](./15-settings.md) | placeholder cards | L |
| 16 | [Infrastructure & Deployment](./16-infrastructure.md) | dev provisioned | M |
| 17 | [Build Roadmap](./17-build-roadmap.md) | n/a | n/a |
| 18 | [SaaS Operations & Tenant Lifecycle](./18-saas-operations.md) | missing | L |
| 19 | [Implementation Backlog & Definition of Done](./19-implementation-backlog.md) | missing | n/a |
| 20 | [First Production Slice — Plan & Status](./20-next-steps.md) | in flight | n/a |
| 21 | [Phase Step 2 — Frontend Learner Module](./21-phase-step-2-frontend-learners.md) | done | n/a |
| 22 | [Deploy Checklist (Steps 1–4 + B2C)](./22-deploy-checklist.md) | pending operator | n/a |
| 23 | [Service Catalog & Driving Centers](./23-service-catalog-and-driving-centers.md) | backend slice in flight | L |
| 24 | [Self-Service School Signup](./24-self-service-signup.md) | planned (post-B2C) | M |

## Short Answer: Does The App Already Have A "Typical School Schedule" Template That Drives Per-Learner Schedule Generation?

**No.** The concept currently has:

- Per-instructor weekly availability (a list of `{weekday, starts_at, ends_at, branch_id}` per instructor â€” see [`dops-api/app/models/instructor.py`](../dops-api/app/models/instructor.py) and the seed at [`dops-api/app/db/seed.py:84`](../dops-api/app/db/seed.py)).
- Concrete `ScheduledSession` rows that admins can create/cancel/reschedule one at a time ([`dops-api/app/models/scheduling.py`](../dops-api/app/models/scheduling.py)).
- A `POST /api/v1/schedule/generate` endpoint that **does not actually generate anything**. The implementation at [`dops-api/app/services/scheduling_service.py:66-70`](../dops-api/app/services/scheduling_service.py) returns `sessions: []` plus a hard-coded suggestion. The Settings UI mentions "Programs", "Session types", and "Cancellation policy" but they are placeholder cards (see [`dops-frontend/src/features/settings/SettingsPage.tsx`](../dops-frontend/src/features/settings/SettingsPage.tsx)).

What is **missing** to support real per-learner generation:

- School-level **operating hours** (typical week) by branch.
- School-level **holiday / closure calendar**.
- **Class period / time-slot** definitions (e.g. theory blocks Mon/Wed 18:00â€“20:00).
- **Program curriculum**: ordered list of theory modules + required practical lessons + evaluations for each program (e.g. Class 5).
- **Learner availability** (windows when a learner can take lessons).
- A real **scheduling engine** that takes (program, learner availability, instructor availability, school operating hours, branch, vehicle, language, location) and produces concrete sessions that satisfy precedence rules between theory/practical/evaluation/road-test-prep.

The full design for this missing feature is in [07-school-schedule-templates.md](./07-school-schedule-templates.md). The companion changes to the existing Scheduling feature are in [06-scheduling.md](./06-scheduling.md). Both are treated as part of the **real implementation of an existing concept feature** â€” not as a new feature â€” because the concept UI already advertises "Generate schedule" buttons, "smart rescheduling", "waitlist slot filling", and "open slot suggestions" that are impossible to implement without these inputs.
