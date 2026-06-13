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

Run the documents in this order. Later docs assume data from earlier docs exists.

| Step | Document | Main outcome |
|---|---|---|
| 1 | [01 - Synthetic data](01-synthetic-data.md) | Prepare accounts and realistic school data |
| 2 | [02 - Auth, onboarding, RBAC](02-auth-onboarding-rbac.md) | Create the tenant and verify role access |
| 3 | [03 - Settings and configuration](03-settings-configuration.md) | Configure the school foundation |
| 4 | [04 - Learners](04-learners.md) | Register learners and complete learner workspace data |
| 5 | [05 - Instructors](05-instructors.md) | Add instructors, availability, credentials, portal access |
| 6 | [12 - Courses and assessments](12-courses-assessments.md) | Author courses, lessons/blocks, quizzes/exams, preview |
| 7 | [13 - Certificate templates and approval](13-certificates-templates.md) | Templates, triggers, review/approve flow |
| 8 | [06 - Scheduling](06-scheduling.md) | Manual + path generation (scenario matrix), attendance, cancellations |
| 9 | [07 - Invoices and payments](07-invoices-payments-certificates.md) | Verify billing and payment surfaces (certificates moved to doc 13) |
| 10 | [08 - Automations, notifications, reviews](08-automations-notifications.md) | WhatsApp/email (auto + manual), document review, messaging |
| 11 | [09 - Public site, leads, reports](09-public-site-reports.md) | Publish the public site, capture leads, check dashboards |
| 12 | [10 - Portals](10-portals.md) | Learner self-service incl. My Courses + certificate; instructor portal |
| 13 | [11 - Platform admin](11-platform-admin.md) | Verify cross-tenant platform admin controls |
| 14 | [15 - Subscriptions: plans and discounts](15-subscriptions-plans-discounts.md) | Platform-admin plan CRUD + discounts (sets AI quotas) |
| 15 | [14 - AI features and token usage](14-ai-features-usage.md) | AI draft replies + per-school/user token metering, quota hard-block |
| 16 | [16 - Mobile app and push](16-mobile-app-push.md) | Expo app: role-locked stacks, attendance, reviews, push notifications |

Note on ordering: docs 14-16 are newest. Run **15 (subscriptions)** before **14 (AI)** because plan
entitlements set the AI token quota that doc 14 exercises. Doc 16 (mobile) reuses data created in the
learner/instructor/scheduling/document docs.

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
| `/learners`, `/learners/new`, `/learners/:id` | Learner list, wizard, workspace | 04 |
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
| `/reports` | Reporting scaffold | 09 |
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

| Area | Current state |
|---|---|
| Learner wizard pickup | Pickup address/notes are collected in the UI, but the submit payload currently does not include them. Test and record as a gap if not persisted. |
| Invoices | API and hooks support create, record payment, refund, and checkout. The visible `/invoices` page lists invoices and a Create invoice dialog; Pay now appears when payouts are configured and surfaces the backend reason (e.g. payouts not connected) on 409. |
| Certificates | Now fully wired: configurable templates (doc 13), four issuance triggers, a clickable certificates list, and a review->approve flow with learner-performance context. Learners view/print their own certificate. Earlier "buttons not wired" note is obsolete. |
| Courses / assessments | Full native LMS: authoring (modules/lessons/blocks/markdown), quizzes/exams with auto-grading + exam gating, preview-as-learner, learner course portal. Auto-grading is exact-match; open-ended questions are out of scope. See doc 12. |
| Schedule rollback | Backend has generation run rollback, but the current Generate dialog exposes preview, regenerate, and commit. Rollback is API/ops coverage unless a UI control appears. |
| Staff reschedule UI | Backend has reschedule endpoints. The current visible session card focuses on attendance and cancellation controls. Record where reschedule is or is not exposed. |
| Automation flow editing | Backend supports updating flows and sending messages. The `/automations` page mostly reads flows/runs/conversations; manual send is mounted on learner/instructor detail. |
| Reports | `/reports` is currently a scaffold with revenue and utilization charts, not a full reporting suite. |
| Documents | Dev storage may be metadata-only. Verify upload/list/review/delete metadata; actual file download may be environment-dependent. |
| WhatsApp / email | In-app rows always created; email needs `email_enabled` (ACS delivery may be a stub on dev); WhatsApp stays `config_pending` until BOTH tenant `whatsappProviderConfigured` and environment credentials are set. See doc 08 channel matrix. |
| Stripe payouts | Online invoice payment requires the school to connect Stripe Connect payouts; until then checkout returns 409 `payouts_not_connected` (surfaced in the UI). |

