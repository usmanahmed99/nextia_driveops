# DrivingOps Manual Test Plan - Overview

This folder is a code-led manual test plan for the current DrivingOps app. Use it page by page,
in order, to set up a real school and then test the operational workflows from the UI.

Primary source for this revision:
- `../dops-frontend/src/app/router.tsx`
- `../dops-frontend/src/lib/routes/routes.ts`
- current page components under `../dops-frontend/src/features`
- current FastAPI endpoints and RBAC in `../dops-api/app/api/v1/endpoints` and
  `../dops-api/app/core/rbac.py`

Product docs can still help explain intent, but when a doc and code disagree, this plan follows
the code.

---

## 1. Test Order

**Start here:** [17 - End-to-End Flow](17-end-to-end-flow.md) is the master journey. It runs the
product top-down the way it is provisioned in production — **platform admin first** (offering +
tenant), then owner, staff, instructors/learners, operations, portals, AI, and the mobile app —
and links each phase to the detail doc that owns the assertions. For a full acceptance pass, drive
from doc 17 and open the feature docs as it points to them. The table below is the same lifecycle in
linear form; later docs assume data from earlier docs exists.

| Step | Phase       | Document                                                                       | Main outcome                                                           |
| ---- | ----------- | ------------------------------------------------------------------------------ | ---------------------------------------------------------------------- |
| 1    | A. Platform | [15 - Subscriptions: plans and discounts](15-subscriptions-plans-discounts.md) | Platform-admin creates the plan (sets AI quota) + discounts            |
| 2    | A. Platform | [11 - Platform admin](11-platform-admin.md)                                    | Create the tenant on that plan, set subscription, act-as               |
| 3    | B. Owner    | [02 - Auth, onboarding, RBAC](02-auth-onboarding-rbac.md)                      | Owner accepts/onboards; verify role access (RBAC sweep)                |
| 4    | B. Owner    | [03 - Settings and configuration](03-settings-configuration.md)                | Configure the school foundation                                        |
| 5    | B. Owner    | [01 - Synthetic data](01-synthetic-data.md)                                    | Prepare realistic accounts/vehicles/categories                         |
| 6    | C. Staff    | (members invites) [02 §02-4..02-7](02-auth-onboarding-rbac.md)                 | Invite + accept office staff; role redirects                           |
| 7    | D. Catalog  | [12 - Courses and assessments](12-courses-assessments.md)                      | Author courses, lessons/blocks, quizzes/exams, preview                 |
| 8    | D. Catalog  | [13 - Certificate templates and approval](13-certificates-templates.md)        | Templates, triggers, review/approve flow                               |
| 9    | E. People   | [05 - Instructors](05-instructors.md)                                          | Add instructors, availability, credentials, portal access              |
| 10   | E. People   | [04 - Learners](04-learners.md)                                                | Register learners and complete learner workspace data                  |
| 11   | F. Operate  | [06 - Scheduling](06-scheduling.md)                                            | Manual + path generation (scenario matrix), attendance, cancellations  |
| 12   | F. Operate  | [07 - Invoices and payments](07-invoices-payments-certificates.md)             | Verify billing and payment surfaces                                    |
| 13   | F. Operate  | [08 - Automations, notifications, reviews](08-automations-notifications.md)    | WhatsApp/email (auto + manual), document review, messaging             |
| 14   | F. Operate  | [09 - Public site, leads, reports](09-public-site-reports.md)                  | Publish the public site, capture leads, check dashboards               |
| 15   | G. Portals  | [10 - Portals](10-portals.md)                                                  | Learner self-service incl. My Courses + certificate; instructor portal |
| 16   | H. AI       | [14 - AI features and token usage](14-ai-features-usage.md)                    | AI draft replies + per-school/user metering, quota hard-block          |
| 17   | I. Mobile   | [16 - Mobile app and push](16-mobile-app-push.md)                              | Expo app: role-locked stacks, attendance, reviews, push notifications  |

Note on ordering: **platform-first.** `POST /platform/tenants` takes a `plan_code` and opens a
trial subscription, so create the real **plan (15)** before the **tenant (11)**; that same plan's
`ai_monthly_tokens` is the quota doc **14 (AI)** later asserts. Doc **16 (mobile)** reuses the
accounts and data created in the web flow, so it runs last.

Note on ordering: docs 12 (courses) and 13 (certificate templates) are authored before 06
(scheduling) because a course can be a learning-path item and its exam can mark a checkpoint that
gates scheduling. If you only want to test scheduling, you can still run 06 first using
service/checkpoint path items and return to the course-gated scenarios afterward.

Keep one master tracking sheet:

| Case ID | Role used | Result | Notes / bug link |
|---|---|---|---|
| 02-1 | Owner | | |

Severity guide: Blocker = cannot continue. Major = feature broken or wrong access. Minor =
cosmetic/copy. RBAC issues are Major or higher.

---

## 2. Current Route Map

These are the current frontend routes to cover.

| Route | Page | Test doc |
|---|---|---|
| `/login` | Login | 02 |
| `/signup` | School signup | 02 |
| `/invite/accept` | Invite acceptance | 02 |
| `/privacy`, `/terms` | Legal documents | 02 |
| `/` | Dashboard | 09 |
| `/onboarding` | Setup checklist | 03 |
| `/settings` and settings subroutes | School configuration | 03 |
| `/learners`, `/learners/new`, `/learners/:id` | Learner list, wizard, workspace (reordered: progress-first) | 04 |
| `/learners/:id/schedule/print` | Printable lesson schedule | 04 |
| `/learners/:id/payments/print` | Printable payment statement (history + upcoming) | 04, 07 |
| `/instructors`, `/instructors/:id` | Instructor list and detail | 05 |
| `/settings/courses`, `/settings/courses/:id/preview` | Course authoring + preview-as-learner | 12 |
| `/settings/certificate-templates` | Certificate template editor | 13 |
| `/schedule` | Calendar, create session, generator | 06 |
| `/invoices` | Invoice list and checkout entry | 07 |
| `/certificates` | Issued certificate columns (clickable) | 13 |
| `/certificates/:id/review` | Certificate review + approve | 13 |
| `/certificates/:id/view` | Certificate print view (office) | 13 |
| `/reviews` | Document review queue | 08 |
| `/automations` | Automation dashboard and notifications panel | 08 |
| `/public-website`, `/public-website/leads` | Site builder and lead inbox | 09 |
| `/site/:slug`, `/site-demo` | Public school site | 09 |
| `/reports` | Reports: revenue/utilization charts + **branch summary** (branch picker, range presets, per-instructor) | 09 |
| `/portal/learner`, `/portal/instructor` | Self-service portals | 10 |
| `/portal/courses`, `/portal/courses/:id` | Learner course catalog + player | 10, 12 |
| `/portal/certificates/:id` | Learner certificate view/print | 10, 13 |
| `/platform/tenants` | Platform tenant management | 11 |

---

## 3. Roles to Test

Use one real email account per role whenever possible.

| Role | Current behavior to verify |
|---|---|
| `PLATFORM_ADMIN` | Can list/manage tenants and act as a tenant. |
| `SCHOOL_OWNER` | Full school access including tenant billing/subscription controls. |
| `SCHOOL_ADMIN` | Broad school admin access, but no tenant-level `tenant.manage` controls. |
| `RECEPTIONIST` | Learner create/update, day-to-day scheduling, document review, reports. |
| `BILLING_SUPPORT` | Invoice/payment work, learner/schedule read, certificates read, reports. |
| `SCHEDULER` | Learner/instructor read and schedule manage/reschedule/cancel, reports. |
| `INSTRUCTOR` | Backend grants own schedule access and reschedule capability; test via instructor portal and own sessions. |
| `LEARNER` | Learner portal, own schedule, own documents, payments/certificates where exposed. |
| `PARENT_GUARDIAN` | Minimal schedule/payment read role; currently no dedicated page in nav. |

Important RBAC notes from code:
- Backend RBAC is the source of truth. Frontend nav is guidance.
- `documents.review` is a current permission. It controls the top-bar review bell and `/reviews`.
- `courses.read` / `courses.manage` gate course viewing vs authoring (Owner/Admin manage; Receptionist read).
- `certificates.read` / `certificates.approve` gate the certificates list vs template editing,
  issuing, and approval (Owner/Admin approve; Receptionist/Billing read).
- Settings -> Members is for office staff roles. Learner and instructor portal invitations are on
  the learner and instructor detail pages.
- Platform admin is not self-serve. It must be seeded or tested through demo mode.

---

## 4. Environment Notes

Auth is Microsoft Entra External ID.

Use the standard sign-in/signup flow with email and password for Gmail `+alias` testing. Do not
use "Continue with Google" for alias accounts; Google treats aliases as the same Google account.

The frontend may point at the deployed dev API unless local `.env` overrides it. If a page uses a
new endpoint and the deployed API is stale, record the environment as blocked instead of filing a
product bug immediately.

Office sidebar order was regrouped by how often each area is used: daily operations (Overview,
Learners, Schedule, Instructors), then money + compliance queues (Invoices, Reviews, Certificates),
then growth + comms (Leads, Public Website, Automations), then insight + assistant (Reports,
Assistant, AI usage), with Settings last. Visibility is still permission-first — the order changed,
not the gating.

Recommended verification order for a feature slice:
1. Sign in with the intended role.
2. Confirm the nav item is present or absent.
3. Open the page normally.
4. Perform the visible UI workflow.
5. Deep-link to restricted routes for RBAC negative checks.
6. Confirm related data appears on downstream pages.

---

## 5. Current UI/API Gaps to Track

These are known from the code inventory and should be treated as current limitations unless the
code changes.

| Area                    | Current state                                                                                                                                                                                                                                                  |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Learner wizard pickup   | Pickup address/notes are collected in the UI, but the submit payload currently does not include them. Test and record as a gap if not persisted.                                                                                                               |
| Invoices                | API and hooks support create, record payment, refund, and checkout. The visible `/invoices` page lists invoices and a Create invoice dialog; Pay now appears when payouts are configured and surfaces the backend reason (e.g. payouts not connected) on 409.  |
| Certificates            | Now fully wired: configurable templates (doc 13), four issuance triggers, a clickable certificates list, and a review->approve flow with learner-performance context. Learners view/print their own certificate. Earlier "buttons not wired" note is obsolete. |
| Courses / assessments   | Full native LMS: authoring (modules/lessons/blocks/markdown), quizzes/exams with auto-grading + exam gating, preview-as-learner, learner course portal. Auto-grading is exact-match; open-ended questions are out of scope. See doc 12.                        |
| Schedule rollback       | Backend has generation run rollback, but the current Generate dialog exposes preview, regenerate, and commit. Rollback is API/ops coverage unless a UI control appears.                                                                                        |
| Staff reschedule UI     | Backend has reschedule endpoints. The current visible session card focuses on attendance and cancellation controls. Record where reschedule is or is not exposed.                                                                                              |
| Automation flow editing | Backend supports updating flows and sending messages. The `/automations` page mostly reads flows/runs/conversations; manual send is mounted on learner/instructor detail.                                                                                      |
| Reports                 | `/reports` has revenue + utilization charts AND a **branch summary** section: branch picker, range presets (today/tomorrow/this week/this month/custom), headline tiles (completed today, revenue today, tomorrow's sessions, unpaid), and a per-instructor table. Same headline tiles also appear on the `/` dashboard. Gated by `reports.read`.                                       |
| Branch-scoped roles     | A staff membership with `branchIds` set is restricted to those branches: learners, sessions, and invoices lists, plus the branch summary, are hard-filtered server-side. Empty/unset `branchIds` = school-wide. Set branches when inviting/editing a member in Settings -> Members. See doc 02 (RBAC) and 03 (members).                                                                |
| Learning-path payment plans | A learning path can define a payment plan (upfront/monthly/quarterly/every-X-months + optional deposit). On enrollment the system builds an installment schedule; a daily job issues each installment invoice on its due date and records an upcoming-payment reminder. `GET /learners/{id}/payment-schedule`. Config in Settings -> Learning paths. See doc 07.                          |
| Missed-session penalties | Each service can define a penalty (none/fixed/percent) + notice window. A no-show always charges it; a cancellation charges it only inside the notice window. The penalty is a standalone invoice linked to the session, waivable by staff (`invoices.manage`). Config in Settings -> Services. See doc 07.                                                                             |
| QC phases / progress detail | Learning-path items can carry a programme phase (e.g. QC's 4 phases) — informational grouping only, no scheduler gating. The learner page shows a progress-detail card: theory/practical done vs remaining (grouped by phase) + payment paid vs remaining, and a printable statement. See doc 04.                                                                                       |
| Scheduler clustering    | The generator now prefers placing an instructor's sessions back-to-back (best-effort adjacency); it never violates availability/closures/caps. Verify generated days cluster rather than scatter when slots allow. See doc 06.                                                                                                                                                          |
| Documents               | Dev storage may be metadata-only. Verify upload/list/review/delete metadata; actual file download may be environment-dependent.                                                                                                                                |
| WhatsApp / email        | In-app rows always created; email needs `email_enabled` (ACS delivery may be a stub on dev); WhatsApp stays `config_pending` until BOTH tenant `whatsappProviderConfigured` and environment credentials are set. See doc 08 channel matrix.                    |
| Stripe payouts          | Online invoice payment requires the school to connect Stripe Connect payouts; until then checkout returns 409 `payouts_not_connected` (surfaced in the UI).                                                                                                    |

