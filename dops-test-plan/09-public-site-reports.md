# 09 - Public Site, Leads, Dashboard, and Reports

Covers public site builder, public anonymous school page, booking leads, dashboard, and current
reports scaffold.

Primary roles:
- Public website builder: Owner/Admin.
- Leads inbox: roles with learner read/create access as enforced by backend.
- Dashboard/reports: roles with `reports.read`.

---

## Current Capability Map

| Area | Current state |
|---|---|
| Public site builder | Edits hero copy, bilingual CTA, visible section toggles, save, publish, preview, live link. |
| Public page | Anonymous `/site/:slug` route and host-based slug resolution. |
| Lead form | Anonymous contact form creates booking leads. |
| Lead inbox | `/public-website/leads` lists leads and moves status. |
| Dashboard | Operational dashboard using learners, schedule, invoices, certificates, instructors, automations. |
| Reports | Revenue + utilization charts, plus a branch summary (branch picker, range presets, headline tiles, per-instructor table). |

---

## 09-1 Public Site Builder Loads

Route: `/public-website`

Roles: Owner/Admin

Steps:
1. Open `/public-website`.
2. Inspect status badge and slug badge.
3. Inspect editor and live preview.

Expected:
- Builder loads from public site preview API.
- Shows draft or published status.
- Shows slug like `{slug}.driveops.com`.
- Editor is seeded with server content.

---

## 09-2 Edit and Save Public Site Content

Roles: Owner/Admin

Steps:
1. Update hero title EN and FR.
2. Update hero subtitle EN and FR.
3. Update CTA label EN and FR.
4. Toggle visible sections: programs, features, reviews, FAQ, instructors.
5. Click Save.
6. Reload builder.

Expected:
- Preview updates as fields change.
- Save persists content.
- Saved indicator appears.
- Reload keeps saved content.

---

## 09-3 Publish and View Live Site

Roles: Owner/Admin

Steps:
1. Click Publish.
2. Click View live.
3. Open `/site/:slug` in a logged-out window.

Expected:
- Publish status changes to published.
- Public page loads without authentication.
- Public site uses school branding, contact data, languages, active programs/services, branches,
  and configured content.
- If site is unpublished or slug is wrong, public page shows unavailable state.

---

## 09-4 Public Site Language Toggle

Roles: public visitor

Steps:
1. Ensure school supports English and French in communication settings.
2. Open `/site/:slug`.
3. Use language toggle.

Expected:
- Toggle is shown only when French is supported.
- Hero, CTA, features, programs, reviews, and FAQ use French fields when available.
- English fallback appears when French copy is missing.

---

## 09-5 Public Lead Form Validation

Roles: public visitor

Steps:
1. Open `/site/:slug`.
2. Submit blank form.
3. Submit with only name but no email/phone.
4. Submit with name and email or phone.
5. Select program interest if options appear.

Expected:
- Form requires full name and at least one way to reach the prospect.
- Valid submission shows thank-you state.
- Public request creates a lead with source `public_site_form`.
- Submission includes preferred language.

---

## 09-6 Lead Inbox

Route: `/public-website/leads`

Roles: Owner/Admin/Receptionist if permitted

Steps:
1. Open lead inbox.
2. Inspect new lead from public form.
3. Click Contacted.
4. Click Converted on a contacted lead.
5. Dismiss a disposable new/contacted lead.

Expected:
- Lead cards show name, email/phone, program interest, message, date, and status.
- Status flow is new -> contacted -> converted.
- Dismiss is available before converted/dismissed.
- Email and phone links use `mailto:` and `tel:`.

---

## 09-7 Lead Notification

Roles: Owner/Admin

Steps:
1. Submit a public lead.
2. Open `/automations`.
3. Inspect notification feed.

Expected:
- Lead notification appears if notification creation succeeds.
- Lead creation does not fail if internal notification creation fails.

---

## 09-8 Public Demo Page

Route: `/site-demo`

Roles: anyone

Steps:
1. Open `/site-demo` logged out.
2. Open logged in.

Expected:
- Demo page renders without authentication.
- It is separate from the real tenant `/site/:slug` public page.

---

## 09-9 Dashboard

Route: `/`

Roles: Owner/Admin/Receptionist/Billing/Scheduler/Platform Admin as permitted

Steps:
1. Open dashboard as Owner.
2. Inspect the headline live tiles: **Completed today**, **Revenue today**, **Tomorrow's sessions**
   (these are computed server-side in the school timezone via the branch-summary endpoint).
3. Inspect the other metrics: active learners, pending payments, pending certificates, etc.
4. Inspect today's sessions, recommendations, blocked work, cancellations, automation health,
   utilization.
5. Repeat as Receptionist, Billing, and Scheduler.

Expected:
- Dashboard loads section by section from real API hooks.
- Completed-today, revenue-today, and tomorrow tiles show real values (collected revenue is the sum
  of succeeded payments today; the old hardcoded "collected MTD" stub is gone).
- Widgets reflect tenant data; limited roles do not crash even if some data is unavailable.

---

## 09-10 Reports + Branch Summary

Route: `/reports`

Roles: roles with `reports.read`

Steps:
1. Open `/reports`. Inspect the revenue chart and utilization chart (names match the instructors
   list).
2. In the Branch summary section choose **All branches** then a specific branch.
3. Cycle the range presets: Today, Tomorrow, This week, This month, Custom (pick custom dates).
4. Inspect the headline tiles (completed today, revenue today, tomorrow, unpaid) and the range
   rollup (sessions in range / completed / no-shows / paid) and the per-instructor table.

Expected:
- The branch summary reflects the selected branch + range; "All branches" aggregates the tenant.
- Headline tiles match the dashboard tiles (09-9) for the all-branches/today view.
- Per-instructor table lists each instructor's total/completed/no-show in the range; empty range
  shows a "no sessions" message.
- For a **branch-scoped** staff member (membership with `branchIds`), the summary is confined to
  their branch(es) and the branch picker cannot widen beyond them (see doc 02 RBAC).

---

## RBAC Negative Checks

### 09-11 Non-admins Cannot Edit Public Site

Roles: Receptionist, Billing, Scheduler, Instructor, Learner

Expected:
- Public website builder nav is hidden unless permission is granted.
- Deep link to `/public-website` is blocked or API returns 403.
- Public `/site/:slug` remains accessible anonymously.

### 09-12 Reports Access

Roles: Instructor, Learner

Expected:
- No Reports/Dashboard staff access unless permissions are changed.
- Deep link to `/reports` is blocked.

