# 02 — Multi-Tenancy, Roles, and Permissions

## Purpose

Every operation in DrivingOps is tenant-scoped. A user can belong to multiple tenants but is always acting **in one tenant at a time**. Authorization is permission-based, not role-based — roles are just bundles of permissions.

## Current State

### Data model

[`dops-api/app/models/tenant.py`](../dops-api/app/models/tenant.py):

- `Tenant(id, slug, status, profile JSON, settings JSON, public_site JSON, audit cols)`
- `Branch(id, tenant_id, name, address, city, province, timezone, phone, audit cols)`

[`dops-api/app/models/user.py`](../dops-api/app/models/user.py) (read in `seed.py`): `User`, `TenantMembership(tenant_id, user_id, role, permissions JSON)`.

### RBAC

[`dops-api/app/core/rbac.py`](../dops-api/app/core/rbac.py) defines:

```python
class Role(StrEnum):
    PLATFORM_ADMIN, SCHOOL_OWNER, SCHOOL_ADMIN, INSTRUCTOR, LEARNER, PARENT_GUARDIAN

class Permission(StrEnum):
    # tenant, users, learners, instructors, schedule, invoices, payments,
    # certificates, automations, public_site, reports, settings,
    # platform.tenants.* …
```

`role_to_permissions(role)` returns the bundle stored on each `TenantMembership`. Endpoints declare requirements with `Depends(require_permission(Permission.X))`.

### Tenant context

[`dops-api/app/core/tenant_context.py`](../dops-api/app/core/tenant_context.py) (verify) reads `X-Tenant-ID` and/or session's active tenant. Each tenant-scoped repo signature takes `tenant_id` and filters in-Python — there is **no DB-level guarantee**.

### Frontend

- Session has `activeTenant`, `memberships`, `permissions` (see `src/lib/contracts/auth.ts`).
- `src/lib/auth/permissions.ts` (verify) defines `hasPermission`, `hasAllPermissions`, `canAccessRoute`.
- `ProtectedRoute`, `PermissionGate` use these.
- Active tenant is also persisted client-side and sent as `X-Tenant-ID` by the HTTP adapter.

## Gaps For Real Implementation

1. **No DB-level enforcement** that every tenant-scoped query includes `tenant_id`. A bug in a service can leak data across tenants.
2. **Membership/permission writes are not exposed**. There is no `POST /tenants/{id}/memberships` or "Staff & permissions" UI; the Settings page only shows a placeholder card.
3. **No support for switching tenants from the UI** for platform admins or multi-tenant staff. (Concept seeds one user per role per tenant; real users will have multiple memberships.)
4. **No tenant-status enforcement**. A `Tenant.status = "suspended"` should fail every request except `auth/session` and platform-admin tenant management; today it is not checked.
5. **Branch authorization** is missing. Instructors should usually be limited to their assigned branch(es); learners likewise. Today `Branch` exists but is not used in any authorization decision.
6. **Subdomain/tenant resolution** is missing. Public site, learner portal, instructor portal should be reachable at `<tenant-slug>.drivingops.ca` and resolve to the correct tenant without `X-Tenant-ID`.
7. **Audit log** exists as a model ([`dops-api/app/models/audit_log.py`](../dops-api/app/models/audit_log.py) — verify), but is not written from services. Every mutating endpoint should emit an `AuditLog(tenant_id, user_id, action, target_type, target_id, before, after, ip, user_agent)` row.

## Real Implementation Plan

### A. Tenant query safety net

In [`dops-api/app/db/base.py`](../dops-api/app/db/base.py) add a `TenantScopedMixin` marker (already present) and a session-level event listener that:

- intercepts every `before_compile` on a Query/Select touching a `TenantScopedMixin` table,
- checks for a `tenant_id == ?` clause,
- raises `RuntimeError("tenant_id filter missing on <Model>")` when absent unless an explicit `with_loader_criteria(..., AdminBypass)` is in scope.

Add a `core/tenant_context.py:tenant_scope(tenant_id)` context manager used by `get_current_tenant_id` and by background jobs.

Tests:

- a repo missing `tenant_id` fails the unit test;
- platform-admin cross-tenant reads work via the explicit bypass token only.

### B. Membership management

Endpoints (new, under existing permissions):

```
GET    /api/v1/tenants/{tenant_id}/memberships          users.manage
POST   /api/v1/tenants/{tenant_id}/memberships          users.manage  body: { user_id|email, role }
PATCH  /api/v1/tenants/{tenant_id}/memberships/{id}     users.manage  body: { role?, permissions?, status? }
DELETE /api/v1/tenants/{tenant_id}/memberships/{id}     users.manage
```

UI: Settings → **Staff & Permissions** card becomes a real page with member list, invite form (email + role), and per-member permission overrides.

### C. Tenant status enforcement

Middleware after auth, before route handlers. If `Tenant.status in {"suspended", "expired"}` and route is not in the allow-list, return `423 Locked` with `{ "code": "tenant_suspended" }`. The frontend `AppShell` renders a full-page "Your school account is suspended — contact billing" state.

### D. Branch-aware authorization

- Add `TenantMembership.branch_ids: list[str] | None`. `None` means all branches.
- Helper `require_branch_access(branch_id)` used on session/instructor list endpoints filtered by branch.
- Learner records have a `branch_id`; instructor records have one default branch + extra allowed branches.

### E. Subdomain resolution

- For prod, the public-site Container App serves `*.drivingops.ca` with a wildcard cert.
- `core/tenant_context.py` resolves tenant by `Host` header for public/portal routes; admin app stays on `app.drivingops.ca` and uses `X-Tenant-ID`.

### F. Audit log

- All write services call `AuditLogger.record(action, target, before, after)` after a successful commit.
- Audit rows are append-only, indexed by `(tenant_id, occurred_at desc)`.
- Settings → **Audit log** card becomes a paginated viewer (read = `tenant.manage` or new `audit.read`).

## Acceptance Criteria

- A unit test that queries a tenant-scoped table without `tenant_id` raises.
- Owner can invite a teammate by email, set role, revoke them; teammate logs in and sees only that tenant's data.
- Suspending a tenant blocks all writes/reads except `auth/session` within one request.
- A platform admin can switch active tenant and see audit logs for it.
- Every write produces exactly one audit-log row (verified by integration test).

## Dependencies

- [03-authentication.md](./03-authentication.md) — needs real user identity before invitations make sense.
- [15-settings.md](./15-settings.md) — surface for staff/branches/audit log management.
