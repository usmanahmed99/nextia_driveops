# 17 — Build Roadmap

Phased plan to turn the concept into the real implementation. Phases are dependency-ordered and each is shippable on its own.

> No new features are introduced. Every item closes a gap from one of the feature docs (00–16).

## Phase 0 — Foundation (clear the demo wiring)

Goal: replace demo-mode toggles and demo auth with real plumbing.

- [03-authentication.md](./03-authentication.md): stand up Azure AD B2C tenant; add JWT verification middleware in API; SPA OIDC flow; keep demo-login behind `ENVIRONMENT=dev`.
- [02-multitenancy-rbac.md](./02-multitenancy-rbac.md): tenant-scope query interceptor; tenant-status middleware; audit logger.
- [18-saas-operations.md](./18-saas-operations.md): tenant lifecycle states, tenant provisioning workflow, onboarding checklist, platform-admin tenant console.
- [16-infrastructure.md](./16-infrastructure.md): OIDC service connection; alert baseline; backups configured.

Definition of done: a real owner account can sign up via B2C, land in an empty tenant, and any unauthenticated call returns 401.

## Phase 1 — Operating data model

Goal: replace JSON-blob columns with first-class tables and re-seed.

- [04-learners.md](./04-learners.md): normalize learners, holds, timeline events.
- [05-instructors.md](./05-instructors.md): normalize instructors, availability rules + exceptions, vehicles, qualifications.
- [07-school-schedule-templates.md](./07-school-schedule-templates.md): schedule templates, operating hours, class periods, closures, programs, program items, cancellation policy.
- [09-invoices-payments.md](./09-invoices-payments.md): invoices/items/payments columns + tax columns.
- [10-certificates.md](./10-certificates.md), [11-documents.md](./11-documents.md): blob path, sha256, audit events.
- [15-settings.md](./15-settings.md): new left-nav layout wired to the new endpoints.

DoD: a school owner can model their school (branches, programs, schedule templates, instructors, vehicles, cancellation policy) entirely from the UI; all data persists; old JSON-blob seed deleted.

## Phase 2 — Scheduling engine

Goal: real per-learner schedule generation.

- [06-scheduling.md](./06-scheduling.md): conflict engine with learner/instructor/vehicle dimension; state machine for session status; cancellation + reschedule reasons; pagination & filters; waitlist + replacement-offer models.
- [07-school-schedule-templates.md](./07-school-schedule-templates.md): `services/scheduling/generator.py` heuristic implementation; dry-run + commit + rollback flow; `schedule_generation_runs` for traceability.

DoD: clicking "Generate schedule" on the Scheduling page for a real tenant with real templates produces a proposal in < 3 s for 50 learners over 30 days, with zero blocking conflicts, and committing creates exactly those sessions.

## Phase 3 — Communications

Goal: outbound automations actually deliver, inbound replies actually arrive.

- [08-automations.md](./08-automations.md): event emission from API; build out `dops-automation-worker`; WhatsApp Cloud API + Azure Communication Services Email transports; webhooks; templates as data; throttling; opt-in/opt-out.
- [02-multitenancy-rbac.md](./02-multitenancy-rbac.md): human-approval queue UI under `automations.manage`.

DoD: every concept flow (onboarding, missing docs, payment followup, smart rescheduling, waitlist, road-test prep, certificate ready, inactive followup, instructor briefing) is dispatched on the correct event, recorded, and visible in the UI; opt-out works; inbound replies update the conversation thread.

## Phase 4 — Money

Goal: real money movement, real receipts, real tax.

- [09-invoices-payments.md](./09-invoices-payments.md): Stripe (Canada) integration; webhook ingress; PDF invoices stored in `documents` container; dunning wired into automations; learner portal payments.
- [16-infrastructure.md](./16-infrastructure.md): webhook routing through Front Door; Stripe secrets in Key Vault.

DoD: a learner pays an invoice from the portal and the receipt appears in their inbox within 30 s; a school admin records cash payment in < 10 s; refunds flow end-to-end.

## Phase 5 — Compliance assets

Goal: real PDFs in storage with proper retention and verification.

- [10-certificates.md](./10-certificates.md): bilingual PDF templates; eligibility check; SAS download URLs; public verification page; revocation flow.
- [11-documents.md](./11-documents.md): SAS upload flow; virus scan; review queue UI; retention policy; ACL by `visibility`.

DoD: a learner uploads a permit, gets scanned and approved; eligible learners get a certificate that a third party can verify via QR.

## Phase 6 — Self-service surfaces

Goal: portals scoped to the real user.

- [13-portals.md](./13-portals.md): `/portal/learner/*` and `/portal/instructor/*` endpoints; portal pages re-wired; cancel/reschedule policy enforced; learner Stripe payments; instructor attendance/lesson-note/availability-exception writes.
- PWA + service worker for offline shell.

DoD: a real learner and a real instructor log in via B2C, see only their own data, and complete a representative day's actions end-to-end.

## Phase 7 — Public site & leads

Goal: each tenant has a live public site driving leads.

- [12-public-site.md](./12-public-site.md): typed sections; public app at `*.drivingops.ca`; bilingual editor; assets; revisions; booking-lead model + Leads UI; SEO basics.
- [16-infrastructure.md](./16-infrastructure.md): wildcard cert + Front Door routing + custom-domain runbook.

DoD: NorthStar (or any tenant) can publish a bilingual site and convert a public lead into a learner with full provenance.

## Phase 8 — Insight

Goal: dashboards and reports back on real data.

- [14-reports-dashboard.md](./14-reports-dashboard.md): aggregate endpoints with snapshots for historic windows; smart-recommendations engine; CSV exports; drill-downs.

DoD: every dashboard number is provable from a SQL query against current data; no hard-coded numbers remain in the UI.

## Phase 9 — Production hardening

Goal: deploy to stg and prod with operational maturity.

- [16-infrastructure.md](./16-infrastructure.md): private networking, Front Door + WAF, identity provider hardened, alerting, backups verified, DR rehearsal.
- [18-saas-operations.md](./18-saas-operations.md): support workflows, subscription entitlements, privacy/data export, release management.
- All apps: load tests, security scan (Trivy + SAST), penetration test pass.

DoD: a real driving school can be onboarded onto prod and serve at least one full month of operations without incident.

## Sequencing Rules

- Phases 0–2 are mandatory and ordered. Skipping any creates blockers downstream.
- Phases 3 and 4 can run in parallel after Phase 2.
- Phase 5 depends on Phase 2 (program completion) + Phase 4 (eligibility includes balance check).
- Phase 6 depends on Phases 3 + 4.
- Phase 7 can start after Phase 0 but its lead-conversion path depends on Phase 1.
- Phase 8 can start at Phase 1 but is most valuable after Phases 2–4.
- Phase 9 is continuous; do not defer to the end.

Use [19-implementation-backlog.md](./19-implementation-backlog.md) as the cross-epic definition of done before promoting any phase from concept to pilot.

## Sizing Estimates

These are rough order-of-magnitude estimates assuming a two-engineer team (one full-stack on frontend+API, one backend+infra). Adjust as needed.

| Phase | Backend | Frontend | Infra/Worker | Weeks |
| ----- | ------- | -------- | ------------ | ----- |
| 0 — Foundation | M | M | M | 3 |
| 1 — Data model | L | L | S | 5 |
| 2 — Scheduling engine | **XL** | L | S | 6 |
| 3 — Communications | L | M | L | 5 |
| 4 — Money | L | M | M | 4 |
| 5 — Compliance | M | M | S | 3 |
| 6 — Portals | M | L | S | 4 |
| 7 — Public site | M | L | M | 4 |
| 8 — Insight | M | M | S | 3 |
| 9 — Hardening | S | S | L | rolling |
