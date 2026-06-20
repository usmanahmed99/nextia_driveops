# 17 - End-to-End Flow: Platform Admin → All Users

This is the **master journey** for a full smoke/acceptance pass. It runs the product top-down,
the way it is actually provisioned in production: a **platform admin** stands up the offering and
the tenant first, the **school owner** onboards and configures, **office staff** are invited to
work, then **instructors** and **learners** are onboarded into their portals, and finally the same
people use the **mobile app**.

Each step links to the detailed feature doc that owns the assertions. This page is the spine; the
feature docs are the muscle. Run them in the order below. Record results in the master tracking
sheet in [00 - Overview](00-test-plan-overview.md).

Code-verified provisioning fact (drives this order): `POST /platform/tenants`
(`tenant_service.create_tenant`) takes `owner_email` + `plan_code`, creates the owner user +
owner membership, and opens a **14-day trial** `TenantSubscription`. If the `plan_code` does not
already exist it auto-creates a stub plan — so creating a **real plan first** (Phase A) is what
makes the tenant land on a properly-entitled plan instead of a placeholder.

---

## Phase A — Platform admin stands up the offering

Role: `PLATFORM_ADMIN` (seeded membership or demo role — platform admin is **not** self-serve).

| Step | Action                                                                                                                                                                                  | Detail doc                                             |
| ---- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| A1   | Sign in as platform admin; confirm `/platform/*` nav is visible and role is `PLATFORM_ADMIN`.                                                                                           | [11](11-platform-admin.md) §11-1, §11-8                |
| A2   | Create the subscription **plan** the school will sell on (e.g. `Growth`, with AI tokens/month set — this sets the AI quota the whole tenant inherits).                                  | [15](15-subscriptions-plans-discounts.md) §15-1..15-3  |
| A3   | Create at least one **discount** (percentage + fixed) and exercise the validation/guards.                                                                                               | [15](15-subscriptions-plans-discounts.md) §15-6..15-10 |
| A4   | **Create the tenant**: slug + school name + owner name + **owner email** + the plan code from A2. Verify invalid slug is rejected and the new tenant appears with a trial subscription. | [11](11-platform-admin.md) §11-3                       |
| A5   | Open **Manage** on the new tenant; confirm subscription shows the A2 plan + trial dates; optionally flip status (trial → active) and confirm it persists.                               | [11](11-platform-admin.md) §11-4, §11-5                |

**Gate before Phase B:** tenant exists, is on the real plan, and an owner email is invited/seeded.

---

## Phase B — School owner accepts, onboards, configures

Role: `SCHOOL_OWNER` (the owner email from A4).

| Step | Action | Detail doc |
|---|---|---|
| B1 | (If platform created an invite) accept the invite as the owner email; otherwise sign in. Land on dashboard/onboarding; confirm full owner nav. | [02](02-auth-onboarding-rbac.md) §02-6, §02-1 |
| B2 | Accept Terms/Privacy consent; confirm consent version is recorded. | [02](02-auth-onboarding-rbac.md) §02-2 |
| B3 | Work the **onboarding checklist**; confirm it reflects real setup state and unblocks as you go. | [02](02-auth-onboarding-rbac.md) §02-3 |
| B4 | Configure the **school foundation**: profile/branding, branches/driving-centers, services, learning paths, roles & policy. | [03](03-settings-configuration.md) |
| B5 | Prepare the realistic **synthetic data set** (accounts/vehicles/categories) used by later phases. | [01](01-synthetic-data.md) |

**Gate before Phase C:** school profile, branches, services, learning paths, vehicles, and license
categories exist. The tenant is now operable.

---

## Phase C — Owner/Admin invites office staff (the team)

Role: `SCHOOL_OWNER` / `SCHOOL_ADMIN` with `users.manage`.

| Step | Action                                                                                                                                                                   | Detail doc                             |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------- |
| C1   | From **Settings → Members**, invite one of each office role: School Admin, Receptionist, Billing Support, Scheduler, Office Manager. Assign branch scope where relevant. | [02](02-auth-onboarding-rbac.md) §02-4 |
| C2   | As **each invited staff member**, accept the invite with the exact invited email and land in the right tenant/role.                                                      | [02](02-auth-onboarding-rbac.md) §02-6 |
| C3   | Verify **login-by-role redirects** for every staff role.                                                                                                                 | [02](02-auth-onboarding-rbac.md) §02-7 |

**Gate before Phase D:** at least Admin, Receptionist, Billing, and Scheduler accounts are active.

---

## Phase D — Build the catalog (courses, certificates)

Role: Owner/Admin (`courses.manage`, `certificates.approve`).

| Step | Action                                                                                                                                                 | Detail doc                         |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------- |
| D1   | Author **courses**: modules/lessons/blocks, quizzes/exams (auto-graded), preview-as-learner. An exam pass can mark a checkpoint that gates scheduling. | [12](12-courses-assessments.md)    |
| D2   | Configure **certificate templates** + the four issuance triggers and the review→approve flow.                                                          | [13](13-certificates-templates.md) |

---

## Phase E — Onboard people: instructors & learners

Roles: Owner/Admin to create; then each invitee.

| Step | Action | Detail doc |
|---|---|---|
| E1 | Add **instructors** (DOB/licence/address/docs), set availability and credentials, and send portal invites from the instructor detail page. | [05](05-instructors.md), [02](02-auth-onboarding-rbac.md) §02-5 |
| E2 | Register **learners** via the wizard, complete the learner workspace, and send learner portal invites. | [04](04-learners.md), [02](02-auth-onboarding-rbac.md) §02-5 |
| E3 | As an instructor and as a learner, **accept the portal invite** and land in the correct portal. | [02](02-auth-onboarding-rbac.md) §02-6 |

**Gate before Phase F:** at least one instructor with availability and several learners on paths.

---

## Phase F — Operate the school (the day-to-day)

Roles: Scheduler/Receptionist/Billing as appropriate.

| Step | Action | Detail doc |
|---|---|---|
| F1 | **Schedule**: manual sessions + path generation (preview→commit, **instructor clustering**), attendance, reschedule/cancellation, vehicle assignment. | [06](06-scheduling.md) |
| F2 | **Invoices & payments**: create invoices, record payment/refund, checkout entry (payouts-gated), **payment-plan installments**, **missed-session penalties + waive**. | [07](07-invoices-payments-certificates.md) |
| F3 | **Automations, notifications, document review, messaging** (in-app always; email/WhatsApp per channel matrix). | [08](08-automations-notifications.md) |
| F4 | **Public site + leads + dashboards + Reports branch summary.** | [09](09-public-site-reports.md) |
| F5 | **Branch-scoped staff:** confirm a member with branch scope sees only their branch across learners/schedule/invoices/branch-summary. | [02](02-auth-onboarding-rbac.md) §02-18 |

---

## Phase G — Self-service portals (learner + instructor on web)

Roles: `LEARNER`, `INSTRUCTOR`.

| Step | Action | Detail doc |
|---|---|---|
| G1 | **Learner portal**: schedule, my path, my courses + course player, documents, certificate view/print, pay-now. Confirm a learner sees **only their own** data. | [10](10-portals.md), [12](12-courses-assessments.md), [13](13-certificates-templates.md) |
| G2 | **Instructor portal**: today/sessions, my-learners (shared-session scoping only — no contact/financial data), availability. | [10](10-portals.md) |

---

## Phase H — AI features (entitlement closes the loop with Phase A)

Role: staff with AI access; platform admin to verify metering.

| Step | Action | Detail doc |
|---|---|---|
| H1 | Use **AI draft replies / assistant**; confirm per-school/per-user token metering. | [14](14-ai-features-usage.md) |
| H2 | Confirm the monthly **quota limit equals the A2 plan's `ai_monthly_tokens`**, and that exhausting it hard-blocks. | [14](14-ai-features-usage.md), [15](15-subscriptions-plans-discounts.md) §15-3 |

---

## Phase I — Mobile app (same users, role-locked)

The mobile app is a thin client over the same dops-api and the **same accounts** created above.
Run after the web flow so there is real data to show. See the dedicated mobile doc for every
assertion: [16](16-mobile-app-push.md).

Preconditions for mobile: app running on the **Pixel 8 DrivingOps** emulator (Metro on port
**8083**), `EXPO_PUBLIC_API_BASE_URL` pointing at the same dops-api/seed data as the web flow,
`EXPO_PUBLIC_ALLOW_DEMO_LOGIN=true` for demo-login (or Entra configured for Microsoft sign-in).
Push registration needs a **physical device**; the emulator covers everything else.

| Step | Action | Detail doc |
|---|---|---|
| I1 | **Auth & branding**: sign-in screen shows the DrivingOps logo; the primary button reads **"Sign in"** (the Microsoft/IdP step is the next screen); demo login routes each role to its own role-locked stack; session persists across relaunch; sign-out clears it. | [16](16-mobile-app-push.md) §16-1..16-4 |
| I2 | **Learner stack**: home/progress, outstanding-balance + Pay now, learning path, documents. | [16](16-mobile-app-push.md) §16-5..16-7 |
| I3 | **Instructor stack**: today + mark attendance (present/no-show, ownership + date enforced), my-learners scoping. | [16](16-mobile-app-push.md) §16-8..16-9 |
| I4 | **Staff stack**: schedule/learners, document reviews (approve/reject), and the **setup-is-web-only** interstitial. | [16](16-mobile-app-push.md) §16-10..16-12 |
| I5 | **Push** (physical device): permission prompt + token registration, receive a push from a session event, graceful fallback, sign-out unregisters. | [16](16-mobile-app-push.md) §16-13..16-16 |

---

## Cross-cutting RBAC sweep (run alongside, not after)

The negative/boundary checks are first-class, not an afterthought. As each role comes online above,
also run its RBAC checks so access is verified at the moment the account exists:

- Per-role web RBAC negatives: [02](02-auth-onboarding-rbac.md) §02-10..02-17.
- Platform isolation (owners are tenant-scoped; non-platform roles cannot reach `/platform/*`):
  [11](11-platform-admin.md) §11-9, §11-10.
- Tenant **status boundary** (suspended tenant limits normal users; platform admin can still
  manage): [11](11-platform-admin.md) §11-7.
- Portal data scoping (learner sees only self; instructor limited to shared-session learners):
  [10](10-portals.md), mirrored on mobile in [16](16-mobile-app-push.md) §16-9.

---

## One-page run order (copy into the tracker)

```
A. Platform admin   → plans (15) → discounts (15) → create tenant (11) → set subscription (11)
B. Owner            → accept/login (02) → consent (02) → onboarding (02) → settings (03) → synthetic data (01)
C. Office staff     → invite members (02-4) → accept (02-6) → role redirects (02-7)
D. Catalog          → courses (12) → certificate templates (13)
E. People           → instructors (05) → learners (04) → portal invites + accept (02-5/02-6)
F. Operations       → scheduling (06) → invoices (07) → automations/reviews (08) → public site/reports (09)
G. Portals (web)    → learner portal (10) → instructor portal (10)
H. AI               → usage/metering (14) ←→ plan quota from A2 (15-3)
I. Mobile           → auth/branding + role stacks + reviews + push (16)
   ↳ RBAC sweep      → 02-10..02-17, 11-7, 11-9, 11-10 run as each role comes online
```
