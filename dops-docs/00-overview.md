# 00 — Product Overview & Personas

## Purpose

DrivingOps is a bilingual (EN/FR) multi-tenant SaaS for Canadian driving schools. Each tenant is a driving school. Each school runs branches, instructors, vehicles, learners, lessons, automations, invoices, and certificates from a single workspace.

The concept demos a fictional tenant: **NorthStar Driving School** (Montreal + Laval, QC).

## Domains The Product Owns

| Domain | Owns |
| ------ | ---- |
| **Identity & Tenancy** | Tenants, branches, users, memberships, roles, permissions, sessions. |
| **Learners** | Learner profile, enrollment in a program, progress, timeline, documents, payment status. |
| **Instructors** | Instructor profile, languages, vehicle, locations, weekly availability, utilization. |
| **Scheduling** | Concrete sessions (theory module, practical lesson, evaluation, road-test prep, admin follow-up), conflicts, cancellations, reschedules, generation, suggestions. |
| **Communications** | Outbound automations on WhatsApp and email, message templates, conversations, escalations to humans. |
| **Billing** | Invoices, payments, balances (no real PSP yet). |
| **Compliance** | Certificates of completion, approval workflow, downloads. |
| **Documents** | Learner permits, consents, other regulated uploads. |
| **Marketing** | Per-tenant public website (subdomain, theme, sections, booking CTA). |
| **Self-service** | Learner portal and instructor portal (mobile-first). |
| **Insights** | Operational dashboard and reports. |

## Personas

| Persona | Role enum | Primary surfaces today |
| ------- | --------- | ---------------------- |
| Platform admin (NextiaAI) | `PLATFORM_ADMIN` | All tenants, platform settings. |
| School owner | `SCHOOL_OWNER` | All school operations + settings, billing, staff. |
| School admin | `SCHOOL_ADMIN` | All school ops except tenant/staff management. |
| Instructor | `INSTRUCTOR` | Instructor portal, today's sessions, schedule view, mark availability. |
| Learner | `LEARNER` | Learner portal, schedule, payments, documents, certificate, support. |
| Parent / guardian (declared, not surfaced) | `PARENT_GUARDIAN` | None today. Permissions list excludes them. |

Permission constants live in [`dops-api/app/core/rbac.py`](../dops-api/app/core/rbac.py).

## Non-Goals For The Real Implementation

These are explicitly **out of scope** for converting the concept into the real implementation (because the concept never showed them):

- Multi-province compliance flows beyond Quebec Class 5 demo.
- Multi-language beyond EN/FR.
- Marketplace (cross-school learner discovery).
- Public learner reviews / ratings.
- Driver simulator integration.
- Fleet/vehicle telematics.
- Mobile native apps (the portals are responsive web).
- Public APIs / partner integrations.

If any of these are needed they should be tracked as new feature work, not as part of the real-implementation effort documented here.

## Branding & Language

- Default language: English. French is a first-class second language.
- All user-facing copy must flow through `src/i18n/{en,fr}/common.json`.
- Bilingual content (templates, public site sections) is stored per-language in the database.
- Tenants can override primary/secondary/accent colors. Currently used by the public site builder and login screen; the rest of the admin UI uses the platform palette.
