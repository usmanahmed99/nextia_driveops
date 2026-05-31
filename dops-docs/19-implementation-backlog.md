# 19 - Implementation Backlog and SaaS Definition of Done

## Purpose

This document turns the feature docs into an executable backlog. It separates must-have production work from later enhancements and gives each area a clear definition of done.

## Product Principles

DrivingOps should be judged by operational outcomes, not screens completed.

The first production-ready release should prove that a driving school can:

1. onboard the school;
2. configure branches, programs, instructors, availability, cancellation rules, and billing rules;
3. register learners;
4. generate and manage schedules;
5. communicate automatically through approved channels;
6. collect money and issue receipts;
7. track progress to certificate;
8. serve learner/instructor portals;
9. operate in English and French;
10. receive support and keep an audit trail.

## Backlog Epics

### Epic 1 - SaaS Foundation

Scope:

- real identity provider;
- tenant provisioning;
- tenant lifecycle statuses;
- membership invitations;
- tenant-scoped query enforcement;
- audit logging;
- platform admin tenant list/detail.

Definition of done:

- no demo auth outside dev;
- every tenant-scoped query is guarded;
- every mutation writes audit log;
- platform admin can provision and suspend a tenant;
- school owner can invite staff.

Primary docs:

- [02-multitenancy-rbac.md](./02-multitenancy-rbac.md)
- [03-authentication.md](./03-authentication.md)
- [18-saas-operations.md](./18-saas-operations.md)

### Epic 2 - School Setup and Configuration

Scope:

- school profile;
- branches;
- operating hours;
- holidays/closures;
- programs;
- session types;
- cancellation policy;
- instructor qualifications;
- tenant onboarding checklist.

Definition of done:

- a new school can configure itself without seed data;
- schedule generation has all required inputs;
- dashboard shows setup guidance until operational setup is complete.

Primary docs:

- [07-school-schedule-templates.md](./07-school-schedule-templates.md)
- [15-settings.md](./15-settings.md)
- [18-saas-operations.md](./18-saas-operations.md)

### Epic 3 - Learner Enrollment and Journey

Scope:

- learner registration wizard;
- guardian support for minors;
- document collection;
- learner holds;
- progress calculation;
- learner workspace endpoint;
- learner timeline events.

Definition of done:

- admin can register a learner end-to-end;
- missing documents/payment holds are explicit records;
- learner detail uses one backend workspace endpoint;
- progress is derived from program/session completion rules.

Primary docs:

- [04-learners.md](./04-learners.md)
- [11-documents.md](./11-documents.md)
- [13-portals.md](./13-portals.md)

### Epic 4 - Instructor Operations

Scope:

- instructor profile;
- branch access;
- vehicle/qualification mapping;
- recurring availability;
- availability exceptions;
- instructor portal actions;
- lesson notes and attendance.

Definition of done:

- instructors see only their assignments;
- admins can model instructor availability accurately;
- lesson completion updates learner progress.

Primary docs:

- [05-instructors.md](./05-instructors.md)
- [06-scheduling.md](./06-scheduling.md)
- [13-portals.md](./13-portals.md)

### Epic 5 - Scheduling Engine

Scope:

- schedule-generation runs;
- dry-run and commit flow;
- conflict detection;
- waitlist;
- replacement offers;
- cancellations/rescheduling;
- instructor/vehicle/location constraints.

Definition of done:

- generate schedule for 50 learners over 30 days in under 3 seconds p95;
- no committed session violates hard constraints;
- every cancellation/reschedule emits events for automations.

Primary docs:

- [06-scheduling.md](./06-scheduling.md)
- [07-school-schedule-templates.md](./07-school-schedule-templates.md)
- [08-automations.md](./08-automations.md)

### Epic 6 - Communications and Automations

Scope:

- event bus;
- worker repo;
- message templates;
- WhatsApp/email provider;
- opt-in/out;
- inbound webhooks;
- human approvals;
- delivery tracking.

Definition of done:

- all core flows dispatch from real events;
- opt-out suppresses future sends;
- inbound replies appear in learner conversation;
- delivery failures become actionable.

Primary docs:

- [08-automations.md](./08-automations.md)
- [18-saas-operations.md](./18-saas-operations.md)

### Epic 7 - Money and Certificates

Scope:

- invoices and tax;
- PSP integration;
- manual payments;
- receipts;
- refunds/credits;
- certificates;
- certificate PDF and verification.

Definition of done:

- learner can pay through portal;
- admin can record cash;
- invoice PDF and receipt are stored;
- certificate eligibility is calculated and auditable.

Primary docs:

- [09-invoices-payments.md](./09-invoices-payments.md)
- [10-certificates.md](./10-certificates.md)
- [11-documents.md](./11-documents.md)

### Epic 8 - Public Website and Lead Conversion

Scope:

- public-site sections;
- public rendering;
- custom tenant subdomain;
- booking/lead form;
- bilingual content;
- lead-to-learner conversion.

Definition of done:

- school can publish a bilingual public site;
- a public lead can be converted into a learner;
- public pages have SEO metadata and acceptable performance.

Primary docs:

- [12-public-site.md](./12-public-site.md)
- [04-learners.md](./04-learners.md)

### Epic 9 - Insights, Reports, and Recommendations

Scope:

- dashboard aggregates;
- operational alerts;
- revenue reporting;
- instructor utilization;
- automation health;
- CSV exports;
- smart recommendations.

Definition of done:

- dashboard numbers are explainable from SQL;
- each recommendation links to an action;
- reports respect tenant/branch permissions.

Primary docs:

- [14-reports-dashboard.md](./14-reports-dashboard.md)

### Epic 10 - Production Readiness

Scope:

- private networking;
- Front Door/WAF;
- custom domains;
- backups and restore drills;
- monitoring and alerts;
- SAST/container scans;
- load testing;
- support operations.

Definition of done:

- production deploy has rollback;
- DR drill is documented;
- critical alerts are tested;
- security scans are required in CI.

Primary docs:

- [16-infrastructure.md](./16-infrastructure.md)
- [18-saas-operations.md](./18-saas-operations.md)

## Cross-Epic Standards

### API Standards

Every production endpoint must include:

- permission dependency;
- tenant isolation;
- Pydantic request/response schema;
- pagination for lists;
- audit logging for writes;
- idempotency support for side effects;
- structured error responses;
- integration tests.

### Frontend Standards

Every production page must include:

- loading state;
- empty state;
- error state;
- permission-aware actions;
- French and English translations;
- mobile layout;
- no direct mock-data imports;
- API-mode verification.

### Database Standards

Every production table must include:

- string UUID primary key or documented natural key;
- `tenant_id` for tenant-owned resources;
- audit timestamps;
- foreign keys where appropriate;
- indexes for common filters;
- migration and rollback notes.

### Event Standards

Every domain event must include:

```json
{
  "eventId": "uuid",
  "eventType": "learner.registered",
  "tenantId": "tenant-id",
  "occurredAt": "ISO-8601",
  "actorUserId": "user-id-or-null",
  "correlationId": "request-id",
  "payload": {}
}
```

Events must be idempotent and safe to replay.

## First Production Slice

Recommended first slice for a real pilot school:

1. Real auth and tenant provisioning.
2. School setup: branches, programs, instructors, availability.
3. Learner registration wizard.
4. Manual scheduling plus conflict detection.
5. Manual invoices and cash/card-placeholder payments.
6. Learner/instructor portals.
7. Email-only automations before WhatsApp.
8. Public site publish and lead form.
9. Dashboard backed by real data.

WhatsApp, full automatic scheduling, and PSP integration can follow after this slice, but the data model should already support them.

## Backlog Readiness Checklist

Before starting an epic:

- API contracts drafted.
- DB migration drafted.
- Frontend UX acceptance criteria written.
- Permission model defined.
- Audit events listed.
- i18n strings listed.
- Failure states defined.
- Seed data updated.
- Test plan agreed.

## Release Readiness Checklist

Before shipping an epic:

- API tests pass.
- Frontend build passes.
- Migration tested on a copy of dev database.
- Rollback command documented.
- App Insights dashboard updated.
- Support notes written.
- Product docs updated.
- Bilingual smoke test completed.

## Risks

| Risk | Mitigation |
|---|---|
| Scheduling engine takes longer than expected | Ship manual scheduling + conflict detection first, then generator |
| WhatsApp policy approval delays launch | Start with email automations; build channel abstraction |
| B2C customization slows auth | Keep OIDC abstraction; allow Auth0/Clerk swap if needed |
| Tenant data isolation bug | Implement tenant query guard before real tenants |
| Payment/tax complexity | Keep school invoices separate from SaaS subscriptions |
| Public site custom domains expand scope | Start with DrivingOps subdomains, custom domains later |

## Non-Goals For First Pilot

- multi-region active-active;
- custom domains for every school;
- fully automatic scheduling with optimization scoring;
- mobile native apps;
- production-grade WhatsApp before opt-in/compliance is complete;
- advanced BI exports beyond CSV.
