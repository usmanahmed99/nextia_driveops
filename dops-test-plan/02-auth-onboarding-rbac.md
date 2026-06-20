# 02 - Authentication, Onboarding, and RBAC

Covers school signup, legal consent, invite acceptance, role redirects, tenant switching, and
negative access checks.

Run this document before operational testing.

---

## 02-1 Owner Signup Creates a School

Roles: new owner account

Steps:
1. Open `/signup`.
2. Use Microsoft Entra External ID sign-up/sign-in with `drivingqa+owner@gmail.com`.
3. Use email and password, not Google social login.
4. Accept the current Terms and Privacy consent if shown.
5. Complete the school form with name, city, phone, and preferred language.

Expected:
- The frontend calls the current school registration flow.
- A tenant is created.
- The user becomes `SCHOOL_OWNER`.
- The app lands on the staff dashboard or onboarding checklist.
- Owner nav includes dashboard, learners, schedule, instructors, automations, invoices,
  certificates, reviews, public website, reports, and settings.

---

## 02-2 Legal Pages and Consent

Roles: any

Steps:
1. Open `/privacy` while logged out.
2. Open `/terms` while logged out.
3. During signup or invite acceptance, open the legal links if present.
4. Try to continue without accepting required consent.

Expected:
- `/privacy` and `/terms` render without authentication.
- Required consent blocks signup/invite acceptance until accepted.
- Consent version is submitted with the onboarding request.

---

## 02-3 Onboarding Checklist Uses Real Setup Data

Roles: Owner

Steps:
1. Open `/onboarding`.
2. Before settings are complete, inspect the checklist.
3. Add profile, driving centers, services, and learning paths in doc 03.
4. Return to `/onboarding`.

Expected:
- The checklist links to `/settings/profile`, `/settings/driving-centers`,
  `/settings/services`, and `/settings/learning-paths`.
- Completion status updates from real API data.
- Continue is disabled until the required setup steps are complete.

---

## 02-4 Invite Office Staff from Settings Members

Roles: Owner or Admin with `users.manage`

Steps:
1. Open `/settings/members`.
2. Invite School Admin, Receptionist, Billing Support, Scheduler, and Office Manager.
3. Assign branch scope where relevant.

Expected:
- The member is created with pending/invited status.
- Settings -> Members offers office/staff roles and custom roles.
- It does not offer learner or instructor profile-linked roles from this form.
- Owner/platform roles are not self-service invite options.

---

## 02-5 Invite Learner and Instructor Portal Users from Detail Pages

Roles: Owner/Admin

Steps:
1. Open a learner detail page.
2. Use the portal access card to invite the learner.
3. Open an instructor detail page.
4. Use the portal access card to invite the instructor, or confirm the create-instructor flow sent the invite.

Expected:
- Learner invitation has `profileType=learner` and the learner profile id.
- Instructor invitation has `profileType=instructor` and the instructor profile id.
- If the profile has no email, the card explains that an email is required.
- Existing invitations can be resent.
- Active memberships show as active.

---

## 02-6 Invitee Accepts Invitation

Roles: each invited user

Steps:
1. Open `/invite/accept?token=...`.
2. Confirm the invite preview shows the school and intended email.
3. Accept Terms/Privacy if prompted.
4. Choose create account for a first-time user, or already have account for an existing user.
5. Sign in/sign up with the exact invited email alias.

Expected:
- The invite preview loads before sign-in.
- The invitation is accepted after authentication.
- The user lands in the intended tenant with the assigned role.
- Email mismatch is rejected.
- Re-opening an already accepted invite does not create duplicate memberships.

---

## 02-7 Login Redirects by Role

Roles: Owner, Admin, Receptionist, Billing, Scheduler, Instructor, Learner

Steps:
1. Sign out.
2. Sign in as each role.

Expected:
- Staff roles land on dashboard or their normal staff home.
- Instructor lands in or is steered to `/portal/instructor`.
- Learner lands in or is steered to `/portal/learner`.
- Already-authenticated users who open `/login` are redirected away from the login page.

---

## 02-8 Demo Role Buttons

Roles: no real account

Steps:
1. Open `/login`.
2. If demo buttons are visible, click each role.

Expected:
- Demo mode only appears when backend login config enables it.
- Each demo role receives the correct nav and permissions.
- Demo mode is useful for smoke testing but does not replace real invitation tests.

---

## 02-9 Tenant Switcher

Roles: user with memberships in two or more tenants

Steps:
1. Invite the same staff email into a second school.
2. Sign in.
3. Use the header tenant switcher.

Expected:
- Active tenant changes.
- All page data reloads for the selected tenant.
- The user's role and permissions are recalculated per tenant.

---

## RBAC Negative Checks

### 02-10 Receptionist

Expected:
- Can access learners, schedule, instructors read views, certificates read, reports, reviews.
- Cannot access settings, automations, invoices, public website builder, platform tenants.
- Deep links to restricted routes are blocked or return 403.

### 02-11 Billing Support

Expected:
- Can access invoices, learner read, schedule read, certificates read, reports.
- Cannot create sessions, cancel as staff, create learners, manage instructors, settings, automations.

### 02-12 Scheduler

Expected:
- Can access learners read, instructors read, schedule manage/reschedule/cancel, reports.
- Cannot access invoices, automations, public website builder, settings.

### 02-13 Instructor

Expected:
- Uses instructor portal and own schedule surfaces.
- Cannot manage sessions as staff.
- Cannot access settings, invoices, automations, platform tenants.
- Backend should not expose a full learner roster through instructor permissions.

### 02-14 Learner

Expected:
- Uses learner portal.
- Cannot access `/learners`, `/instructors`, `/settings`, `/reports`, `/automations`, or staff schedule management.
- Cannot read another learner's data by deep-linking.

### 02-15 School Admin

Expected:
- Broad school management access.
- No tenant-level `tenant.manage` actions such as owner-only billing/subscription controls.

### 02-16 Platform Admin

Expected:
- Sees `/platform/tenants`.
- Can act as tenants.
- Non-platform roles cannot deep-link to `/platform/tenants`.

### 02-17 Logout

Steps:
1. Use account menu -> sign out.
2. Open a protected route.

Expected:
- Session is cleared.
- User is sent to `/login`.
- Protected routes no longer load data.

### 02-18 Branch-Scoped Staff Are Confined to Their Branch(es)

Roles: Owner/Admin to configure; the scoped member to verify

Preconditions: the tenant has at least two branches and learners/sessions/invoices in each.

Steps:
1. In Settings -> Members, edit (or invite) a staff member and set their branch scope to ONE branch
   (the membership's `branchIds`). Leave another comparable member school-wide (no branches).
2. Sign in as the branch-scoped member. Open Learners, Schedule, Invoices, and Reports -> Branch
   summary.
3. Deep-link / API: request a learner, session, or invoice that belongs to the OTHER branch; request
   the branch summary for the other branch.

Expected:
- Learners, Schedule, and Invoices lists show only the scoped branch's records.
- The branch summary is confined to the scoped branch; the branch picker cannot widen beyond the
  member's branch(es), and asking for a foreign branch returns 403 (or empty), not other-branch data.
- The school-wide member still sees all branches.
- Empty/unset `branchIds` means school-wide (the prior default); existing members are unaffected
  until a scope is set.
