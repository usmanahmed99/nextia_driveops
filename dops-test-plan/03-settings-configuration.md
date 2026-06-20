# 03 - Settings and Configuration

Covers the current Settings area and onboarding checklist. Complete this before testing learners,
instructors, scheduling, billing, public site, or portals.

Primary roles: School Owner and School Admin.

Negative roles: Receptionist, Billing Support, Scheduler, Instructor, Learner.

---

## 03-1 Settings Landing Page

Route: `/settings`

Steps:
1. Open `/settings` as Owner.
2. Open `/settings` as Admin.
3. Inspect every card.

Expected:
- Cards link to current routes: profile, account, branding, communication, policy, license
  categories, checkpoints, opening hours, vehicles, closures, members, roles, driving centers,
  services, learning paths, **courses**, **certificate templates**, billing, and payouts.
- Branches routes to driving centers.
- Permissions routes to roles.
- WhatsApp/language routes to communication.
- Courses routes to `/settings/courses` (covered in doc 12).
- Certificate templates routes to `/settings/certificate-templates` (covered in doc 13).
- Programs and sessions cards are present as placeholders only if still shown; they should not
  be treated as working settings pages unless a route exists.

---

## 03-2 Profile Settings

Route: `/settings/profile`

Steps:
1. Enter the profile data from doc 01.
2. Clear each required field one at a time.
3. Save valid data.
4. Reload the page and open the header/site builder.

Expected:
- Required fields include name, address, city, province, phone, email, and timezone.
- Invalid/missing required values block save.
- Valid save updates the tenant profile and refreshes the active session.
- School name and branding-dependent pages reflect the update.

---

## 03-3 Account Settings

Route: `/settings/account`

Steps:
1. Open as a staff user.
2. Set a user timezone override.
3. Clear the override to use school timezone.
4. Save and reload.

Expected:
- User timezone persists through `PATCH /users/me`.
- Blank timezone falls back to the school timezone.
- This is per-user, not school-wide.

---

## 03-4 Branding

Route: `/settings/branding`

Steps:
1. Enter the logo URL and three hex colors from doc 01.
2. Change each color with valid `#RRGGBB` values.
3. Try invalid values such as `12345` and `#zzzzzz`.
4. Save and reload.

Expected:
- Swatches/preview update.
- Valid values persist.
- Invalid hex values are blocked.
- Public site preview uses the saved palette.

---

## 03-5 Communication and Languages

Route: `/settings/communication`

Steps:
1. Enable English and French.
2. Set default language to English.
3. Enable email, WhatsApp, and WhatsApp opt-in required.
4. Try to remove every supported language.
5. Try to set a default language that is not supported.
6. Save and reload.

Expected:
- At least one language is required.
- Default language must be included in supported languages.
- Email and WhatsApp toggles persist.
- WhatsApp enabled but unconfigured later produces a notification warning in doc 08.

---

## 03-6 Policy Settings

Route: `/settings/policy`

Steps:
1. Set cancellation window to 24 hours.
2. Set invoice due days to 14.
3. Enable certificate approval required.
4. Try boundary values: cancellation `0`, `720`; invoice due `0`, `365`.
5. Try invalid values outside those ranges.
6. Save and reload.

Expected:
- Valid ranges persist.
- Invalid ranges are rejected.
- Cancellation and invoice policies are reflected downstream where the UI/API uses them.

---

## 03-7 Driving Centers

Route: `/settings/driving-centers`

Steps:
1. Create Downtown Centre as `school_branch`.
2. Create North York Centre as `school_branch`.
3. Create Metro SAAQ Test Centre as `saaq_service_center`.
4. Edit phone/email/notes for one center.
5. Deactivate one center, then inspect inactive list.

Expected:
- Supported types include school branch, SAAQ service center, partner center, office, and other.
- School branch/office are owned by the tenant.
- Centers appear in branch, vehicle, learner enrollment, and session location selectors where applicable.
- Deactivation hides inactive centers from normal active selectors.

---

## 03-8 Services

Route: `/settings/services`

Steps:
1. Create the services from doc 01 with code, name, optional French name, kind, duration, buffer,
   default location type, fee, currency, tax code, and vehicle/instructor requirements.
2. Try an invalid code with spaces.
3. Try duration below 5 or above 600.
4. Edit service fee and buffer.
5. Deactivate one service and inspect create-session and learning-path selectors.
6. In the Missed-session penalty section choose a penalty type: None, Fixed amount, or Percentage of
   fee. For Fixed set the amount; for Percentage set the percent; set the notice window (hours). Save
   and reopen.

Expected:
- Codes only accept valid code characters.
- Duration and buffer validation match the UI limits.
- Active services appear in learning paths and Create Session.
- Service fields drive scheduling defaults: duration, buffer, location, vehicle requirement,
  instructor requirement, and fee.
- Prerequisite/checkpoint fields persist if configured.
- Penalty config persists: type + amount/percent + notice hours. Selecting "None" hides the amount/
  percent/notice inputs. The penalty drives auto-charging on no-show/late-cancel (see doc 07-14/15).

---

## 03-9 License Categories

Route: `/settings/license-categories`

Steps:
1. Create the four categories from doc 01.
2. Toggle `class_4_taxi` inactive.
3. Try duplicate code `class_5_auto`.
4. Delete a disposable category.

Expected:
- Active categories appear in learner enrollment, instructor categories, and vehicle categories.
- Inactive categories are visibly inactive and excluded from active selectors.
- Duplicate codes are rejected.
- Delete/deactivate controls do not affect unrelated categories.

---

## 03-10 Checkpoints

Route: `/settings/checkpoints`

Steps:
1. Create `learner_permit_obtained`, `mid_course_review`, and `final_school_approval`.
2. Toggle one inactive.
3. Try duplicate code.
4. Delete a disposable checkpoint.

Expected:
- Active checkpoints appear as learning-path rows.
- Duplicate codes are rejected.
- Inactive/deleted checkpoints are not offered for new path rows.

---

## 03-11 Learning Paths

Route: `/settings/learning-paths`

Preconditions:
- At least one active service exists.
- At least one active checkpoint exists.

Steps:
1. Create `Class 5 Auto Beginner`.
2. Set kind, license category, package price, and discount.
3. Add ordered **service**, **course**, and **checkpoint** rows. The item-type dropdown offers
   Service / Course / Checkpoint; for a Course row, pick a published course (drafts are flagged).
4. Configure timing/gating details: min start week, depends-on sequence, min days after dependency,
   gate service, gate checkpoint, and gate required.
5. Per item, set a **Phase** (No phase / Phase 1-4) for QC's 4-phase grouping.
6. In the **Payment plan** section pick a mode (Pay upfront / Monthly / Quarterly / Every X months);
   for non-upfront set installments (+ interval months) and an optional deposit.
7. Reorder rows by drag/drop.
8. Save, reload, deactivate, reactivate.

Expected:
- Create path is disabled or blocked until services exist.
- Service, course, and checkpoint rows persist in order.
- A course row that has a completion certificate template shows an "Issues certificate: <name>" hint.
- A checkpoint row that has a certificate template shows the same hint (the learning-path milestone
  certificate surface).
- Package price and discount persist.
- Rows support counts and notes.
- Per-item phase persists and later groups the learner's progress detail (doc 04-20); it does NOT
  gate scheduling (informational only).
- The payment plan persists; non-upfront modes show installments/interval/deposit inputs, upfront
  hides them. Assigning the path to a learner builds the installment schedule (doc 07-12).
- Timing and gate fields persist.
- Path filters by license category in learner enrollment.
- Active paths are usable in learner registration and schedule preview.

> Full course-as-path-item coverage (the course->exam->checkpoint->gate chain) is in
> [12 - Courses and Assessments](12-courses-assessments.md), case 12-8, and the scheduling impact in
> [06 - Scheduling](06-scheduling.md), Scenario G.

---

## 03-12 Opening Hours

Route: `/settings/opening-hours`

Steps:
1. Enter the weekly hours from doc 01.
2. Disable Sunday.
3. Try an enabled day with end time before start time.
4. Save and reload.

Expected:
- Enabled days require a valid start/end range.
- Disabled days do not require times.
- Saved hours reload.
- Scheduling/generation should respect school hours where implemented.

---

## 03-13 Closures

Route: `/settings/closures`

Steps:
1. Create all-branch Canada Day.
2. Create branch-specific Summer maintenance.
3. Create timed Staff meeting.
4. Try a timed closure with end time before start time.
5. Delete one closure.

Expected:
- All-day and timed closures persist.
- Branch-specific and all-branch scopes are clear.
- Invalid time ranges are rejected.
- Generator does not place sessions on closure windows.

---

## 03-14 Vehicles

Route: `/settings/vehicles`

Steps:
1. Add Toyota Corolla and Honda Civic as school-owned vehicles.
2. Add Mazda 3 as instructor-owned vehicle.
3. Try school ownership without branch.
4. Try instructor ownership without instructor.
5. Assign Toyota to Ivan for a date range.
6. Add a Honda blackout.
7. Delete the assignment and blackout.

Expected:
- School-owned vehicles require a branch.
- Instructor-owned vehicles require an instructor.
- UI does not require year because the current form does not expose it.
- Categories/transmission persist.
- Assignments and blackouts persist and can be removed.
- Scheduling/generator respects category, transmission, assignment, and blackout where applicable.

---

## 03-15 Members

Route: `/settings/members`

Steps:
1. Invite School Admin, Receptionist, Billing, Scheduler, and Office Manager.
2. Edit a member role and branch scope.
3. Resend an invite.
4. Revoke a pending invite.
5. Open the member documents panel and upload a staff document.
6. Edit communication preferences if exposed.

Expected:
- Members page is for staff/office roles.
- Instructor and learner profile invites are handled on their detail pages, not here.
- Resend/revoke work for pending invitations.
- Role changes apply on the user's next session.
- Member documents are scoped to the selected member.

---

## 03-16 Roles

Route: `/settings/roles`

Steps:
1. Open Roles.
2. Inspect system roles.
3. Create Office Manager with permissions from doc 01.
4. Invite a member using Office Manager.
5. Add another permission to Office Manager.
6. Sign out/in as that member.
7. Delete a disposable custom role.

Expected:
- System roles are read-only.
- Custom roles can be created, edited, and deleted.
- Permission changes resync to memberships on a new session.
- System roles cannot be deleted.

---

## 03-17 Billing Usage

Route: `/settings/billing`

Roles: Owner first, then Admin

Steps:
1. Open as Owner.
2. Inspect active/archived/monthly usage metrics.
3. Start checkout if a subscription checkout button is available.
4. Open billing portal if configured.
5. Open as Admin.

Expected:
- Owner can access tenant-level billing/subscription actions.
- Admin may view settings but tenant-level controls should be hidden, disabled, or API-forbidden
  where they require `tenant.manage`.
- Stripe actions may be blocked by environment configuration; record as environment blocked if so.

---

## 03-18 Payouts

Route: `/settings/payouts`

Roles: Owner/Admin if route is visible

Steps:
1. Open payouts page.
2. Inspect Connect status.
3. Click Connect payouts or Resume if present.
4. If connected and charges enabled, click Open dashboard.

Expected:
- Status loads from billing payouts API.
- Missing Stripe Connect configuration is reported cleanly.
- Onboarding/dashboard links navigate only when URLs are returned.
- Payment checkout later depends on payouts being connected and charges enabled.

---

## 03-19 Onboarding Checklist Completion

Route: `/onboarding`

Steps:
1. Complete profile, driving centers, services, and learning paths.
2. Return to `/onboarding`.

Expected:
- Checklist shows each step complete.
- Continue becomes available.
- No stale placeholder setup step remains blocking progress.

---

## RBAC Negative Checks

### 03-20 Non-admin Roles Cannot Configure Settings

Roles: Receptionist, Billing, Scheduler, Instructor, Learner

Steps:
1. Confirm Settings nav is hidden.
2. Deep-link to `/settings`, `/settings/roles`, `/settings/vehicles`, `/settings/billing`.

Expected:
- Routes are blocked or redirect.
- API calls return 403.
- No settings data is leaked.

