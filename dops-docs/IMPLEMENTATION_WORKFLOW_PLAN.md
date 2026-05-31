# DrivingOps Implementation Workflow Plan

This plan is based on the corrected product docs folder at `../dops-docs`. It translates the roadmap into the natural setup order we should keep implementing and verifying.

## Current Baseline

- Login is working and has been manually verified.
- The docs say the first production slice has already moved through auth foundation and backend learner normalization work.
- The next best product step is not another isolated page. It is a coherent owner onboarding path: configure the school, then the team, then services and learning paths, then learners, then attendance/check-in.

## Working Rhythm

Each implementation step should finish with:

- backend contract and persistence in place;
- frontend page or workflow wired to API mode;
- loading, empty, error, permission, and bilingual states handled;
- seed data updated for dev verification;
- audit/event/idempotency behavior added for writes where appropriate;
- local tests/builds run;
- one manual smoke path documented.

## Priority 1 - School Setup Foundation

Goal: a school owner can configure the school without relying on seed data.

Scope:

- Settings left-nav structure from `15-settings.md`.
- Profile: legal name, contact, address, default language, supported languages.
- Branding: logo, theme colors, public-facing brand defaults.
- Branches and locations: at least one branch, address, timezone, active state.
- Onboarding checklist state: show setup guidance until required setup is complete.

Verification:

- Owner logs in, updates profile and branding, reloads, and sees persisted values.
- Owner creates or edits a branch and cannot remove the final active branch.
- School setup progress reflects completed sections.

Primary docs:

- `../dops-docs/15-settings.md`
- `../dops-docs/12-public-site.md`
- `../dops-docs/18-saas-operations.md`

## Priority 2 - Team, Instructors, Vehicles

Goal: the school can model the people and assets needed before scheduling.

Scope:

- Staff and permissions surface in Settings.
- Instructor profile normalization: name, contact, branches, preferred languages, status.
- Vehicles: branch, make/model/year, transmission, plate, accessibility, service status.
- Qualifications and instructor-to-vehicle assignment.
- Availability rules and exceptions, including effective availability preview.
- Optional `slot_gap_minutes` override per instructor.

Verification:

- Owner invites or creates a staff member/instructor.
- Admin assigns instructor branches, qualifications, vehicle, and weekly availability.
- A sick-day or day-off exception removes the instructor from effective availability.

Primary docs:

- `../dops-docs/05-instructors.md`
- `../dops-docs/02-multitenancy-rbac.md`
- `../dops-docs/23-service-catalog-and-driving-centers.md`

## Priority 3 - Services, Programs, Learning Paths

Goal: the school can define what it sells and what each learner must complete.

Scope:

- Driving centers: school branches and external test centers.
- Service offerings: lesson, evaluation, road test, admin service, duration, buffer, fee, default location, vehicle/instructor requirements.
- Programs: bilingual names, price, required theory/practical/evaluation counts.
- Program items: ordered curriculum, durations, prerequisites, archival rules.
- Schedule templates: operating hours, closures, class periods, default slot gap.
- Cancellation policy.

Verification:

- Owner creates a service with a duration, fee, buffer, and default location.
- Owner creates a program and ordered learning path.
- Owner previews a schedule template and sees closures and operating hours applied.

Primary docs:

- `../dops-docs/23-service-catalog-and-driving-centers.md`
- `../dops-docs/07-school-schedule-templates.md`
- `../dops-docs/06-scheduling.md`

## Priority 4 - Learner Enrollment

Goal: admins can register learners only after school setup prerequisites exist: profile, locations, services, and learning paths/packages.

Scope:

- Register learner wizard: identity, DOB, language, contact, branch, learning path/package, pickup details, documents, payment plan placeholder, confirmation.
- Learner availability windows captured during registration.
- Holds: payment, documents, medical, behavior, admin review, other.
- Learner workspace as the single detail-page source.
- Instructor/program assignment and transfer/withdraw workflows.

Verification:

- Admin registers a learner end to end only after setup prerequisites are complete, then sees the learner in the server-filtered list.
- Admin opens learner workspace and sees journey, holds, upcoming sessions, documents, invoices, and recommended actions.
- Adding a payment or document hold changes list badges and blocks scheduling candidacy.

Primary docs:

- `../dops-docs/04-learners.md`
- `../dops-docs/11-documents.md`
- `../dops-docs/09-invoices-payments.md`

## Priority 5 - Manual Scheduling And Attendance

Goal: staff can schedule, run, and close the day before full auto-generation.

Scope:

- Service-aware manual session creation.
- Bounded sessions list with filters by date, branch, instructor, learner, status.
- Conflict checks for learner, instructor, vehicle, operating hours, and closures.
- Session status state machine.
- Cancel, reschedule, complete, no-show, and check-in actions with reasons.
- Attendance record and lesson notes on completion.
- Learner progress updates from completed sessions.

Verification:

- Admin creates a session from a service; duration, fee, location, and buffer prefill.
- Instructor or admin checks in a learner, marks attendance, adds lesson notes, and completes the session.
- No-show and cancellation persist reason/history and emit events.
- Completed practical/theory sessions update learner progress.

Primary docs:

- `../dops-docs/06-scheduling.md`
- `../dops-docs/13-portals.md`
- `../dops-docs/08-automations.md`

## Priority 6 - Schedule Generation

Goal: use the already configured school model to generate safe schedule proposals.

Scope:

- Dry-run `POST /schedule/generate` using template, program items, learner availability, instructor effective availability, vehicles, services, holds, and closures.
- Generation run records with proposal, commit, rollback, and decision log.
- Review UI grouped by day, learner, conflict, and suggestion.
- Commit creates exactly the proposed sessions.

Verification:

- Generate a 30-day proposal for a small pilot cohort.
- Confirm no hard conflicts in committed sessions.
- Rollback removes only sessions created by the run.

Primary docs:

- `../dops-docs/06-scheduling.md`
- `../dops-docs/07-school-schedule-templates.md`

## Priority 7 - Communications, Money, Certificates, Portals

Goal: complete the operating loop after setup, learners, scheduling, and attendance are reliable.

Scope:

- Email-first automations for booking, cancellation, reschedule, missing docs, payment follow-up, and certificate ready.
- Manual invoices and payment recording first, PSP later.
- Certificate eligibility and PDF generation after completion and balance rules.
- Learner and instructor portals for scoped self-service.

Verification:

- Events appear in the outbox and worker logs.
- Payment hold and document hold workflows are visible in learner workspace.
- Instructor sees only assigned sessions and can complete attendance/notes.
- Learner sees own schedule, requirements, and payment/document tasks.

Primary docs:

- `../dops-docs/08-automations.md`
- `../dops-docs/09-invoices-payments.md`
- `../dops-docs/10-certificates.md`
- `../dops-docs/13-portals.md`

## Keep Continuous

These should be improved alongside every step, not saved for the end:

- tenant isolation and RBAC;
- audit logs and domain events;
- bilingual UI strings;
- observability and App Insights events;
- support notes and runbooks;
- seed data that exercises the latest workflow;
- API mode verification from the frontend.

## Next Concrete Slice

Start with Priority 1:

1. Audit current Settings API and frontend state.
2. Implement or complete `GET/PUT /settings/profile`.
3. Implement or complete `GET/PUT /settings/branding`.
4. Convert Settings from placeholder cards to left-nav sections for Profile, Branding, Branches, Programs, Services, Team, and Policies.
5. Wire Profile and Branding first.
6. Verify owner can update profile/logo/theme, reload, and keep working in API mode.
