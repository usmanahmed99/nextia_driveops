# 11 - Platform Admin

Covers the current cross-tenant platform tenant page and act-as behavior.

Role: `PLATFORM_ADMIN`.

Platform admin is not self-serve. Use a seeded membership or demo role if available.

There is no email-domain auto-elevation. Signing in with a Nextia-like email does not grant
platform admin by itself.

Preconditions:
- At least two tenants exist.
- Platform admin account has `platform.tenants.read` and `platform.tenants.manage` if testing writes.

---

## 11-1 Platform Tenants Nav

Route: `/platform/tenants`

Steps:
1. Sign in as platform admin.
2. Inspect sidebar.
3. Open `/platform/tenants`.

Expected:
- Platform tenants nav item is visible only for platform admin permissions.
- Page loads all tenants.
- Non-platform users do not see this route.

---

## 11-2 Tenant List Details

Steps:
1. Inspect each tenant card.

Expected:
- Each card shows school name, slug, status badge, city/province, and public-site subdomain.
- Active tenant is marked.
- Tenants other than active tenant show Switch to action.
- Manage action opens tenant manage dialog.

---

## 11-3 Create Tenant

Roles: Platform admin with `platform.tenants.manage`

Steps:
1. Click Create tenant.
2. Enter slug, school name, owner name, owner email, and pick a **plan from the dropdown**
   (no free-text plan code; it lists real plans by name + code).
3. Try invalid slug with spaces.
4. Submit valid data.

Expected:
- Invalid slug is blocked.
- The plan dropdown defaults to the first plan; with zero plans it shows
  "No plans available — create one first" and submit is disabled.
- Valid create creates a tenant and owner membership/invite according to backend behavior.
- New tenant appears in the list.
- Read-only platform users do not see create action.

---

## 11-4 Manage Tenant Status

Roles: Platform admin with manage permission

Steps:
1. Open Manage for a disposable tenant.
2. Change status between trial, active, past_due, suspended, cancelled, archived, or provisioning.
3. Save.
4. Refresh.

Expected:
- Status persists.
- Status badge updates.
- Read-only platform users can inspect but cannot save status.

---

## 11-5 Manage Tenant Subscription

Roles: Platform admin with manage permission

Steps:
1. Open Manage for a tenant.
2. Inspect subscription fields.
3. Change the **plan** (dropdown), subscription status, **discount** (dropdown), billing provider,
   and billing customer id.
4. Save.

Expected:
- Subscription loads; plan and discount are dropdowns (not raw UUID text boxes).
- Trial/period dates are shown when available.
- Changes persist after refresh.
- The dialog also shows a **Plan usage** panel and an **Extra quota (overrides)** section —
  see doc 15 (15-13 … 15-17) for the discount-apply, usage, and override cases.
- Read-only platform users cannot save.

---

## 11-6 Act As Tenant

Roles: Platform admin

Steps:
1. Click Switch to on a tenant that is not active.
2. Open dashboard, learners, settings, and schedule.
3. Switch to another tenant from header or tenant page.

Expected:
- Active tenant changes.
- App data is scoped to the selected tenant.
- Platform admin has owner-level access while acting as a tenant, plus platform permissions.

---

## 11-7 Tenant Status Boundary

Roles: Platform admin and normal owner

Steps:
1. Suspend a disposable tenant.
2. Sign in as that tenant's normal owner.
3. Sign in as platform admin and manage the tenant.

Expected:
- Normal tenant users see blocked/limited suspended-tenant UI where boundary is implemented.
- Platform admin can still open platform management and change status back.

---

## 11-8 Platform Admin Session Verification

Steps:
1. Inspect `/auth/session` response if API testing.

Expected:
- Role is `PLATFORM_ADMIN`.
- Permissions include `platform.tenants.read` and `platform.tenants.manage` for full platform admin.
- Membership permissions derive from role unless explicitly customized.

---

## RBAC Negative Checks

### 11-9 Non-platform Roles Cannot Access Platform Tenants

Roles: Owner, Admin, Receptionist, Billing, Scheduler, Instructor, Learner

Steps:
1. Confirm no Platform tenants nav.
2. Deep-link to `/platform/tenants`.

Expected:
- Route is blocked or API returns 403.
- Non-platform users cannot list all tenants.
- Single-tenant owners see only their own tenant context, not platform switcher/list.

### 11-10 Owner Is Tenant-Scoped

Roles: School Owner

Steps:
1. Sign in as owner.
2. Try to list platform tenants through UI or API.
3. Try to act as another tenant.

Expected:
- Owner can manage only own tenant.
- Cross-tenant platform APIs return 403.

