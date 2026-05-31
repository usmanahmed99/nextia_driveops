# 18 - SaaS Operations, Tenant Lifecycle, and Commercial Readiness

## Purpose

DrivingOps is not only a school-management app. It is a multi-tenant SaaS business. This document defines the operational layer needed to sell, onboard, support, bill, monitor, suspend, recover, and grow real Canadian driving-school tenants.

## Current State

The product has:

- a tenant model;
- demo roles and RBAC;
- a polished admin UI;
- deployed dev infrastructure;
- placeholder settings and billing cards;
- no real tenant onboarding, plan management, subscription billing, support tooling, or platform admin operations.

The current `PLATFORM_ADMIN` role exists, but there is no meaningful platform console beyond placeholder navigation.

## Tenant Lifecycle

DrivingOps needs explicit lifecycle states:

```text
lead -> trial_requested -> trial_active -> active -> past_due -> suspended -> cancelled -> archived
```

Recommended tenant statuses:

| Status | Meaning | Allowed Operations |
|---|---|---|
| `provisioning` | Tenant is being created; seed/setup still running. | Platform admin only |
| `trial` | Tenant can use product with trial limits. | Normal operations, limits enforced |
| `active` | Paid or approved tenant. | Full operations |
| `past_due` | Payment failed or invoice overdue. | Read allowed, writes optionally limited |
| `suspended` | Account locked for non-payment, abuse, or admin action. | Login/session only, no tenant operations |
| `cancelled` | Tenant ended subscription. | Owner export-only window |
| `archived` | Tenant data retained but not active. | Platform admin only |

Required backend fields:

```sql
tenants
  ADD COLUMN lifecycle_status text not null default 'trial',
  ADD COLUMN trial_started_at timestamptz null,
  ADD COLUMN trial_ends_at timestamptz null,
  ADD COLUMN activated_at timestamptz null,
  ADD COLUMN suspended_at timestamptz null,
  ADD COLUMN suspended_reason text null,
  ADD COLUMN cancelled_at timestamptz null,
  ADD COLUMN cancellation_reason text null,
  ADD COLUMN data_retention_until timestamptz null;
```

## Tenant Provisioning

Tenant creation should be a workflow, not a direct insert:

1. Create `Tenant`.
2. Create default branch.
3. Create owner membership.
4. Create default roles/permissions.
5. Create default program templates.
6. Create default communication templates in English and French.
7. Create default public-site draft.
8. Create trial subscription.
9. Emit `tenant.provisioned`.

Endpoint:

```text
POST /api/v1/platform/tenants
permission: platform.tenants.manage
```

Payload:

```json
{
  "schoolName": "NorthStar Driving School",
  "ownerEmail": "owner@example.com",
  "province": "QC",
  "languages": ["en", "fr"],
  "defaultBranch": {
    "name": "Montreal",
    "timezone": "America/Toronto"
  },
  "planCode": "starter"
}
```

Acceptance criteria:

- Tenant provisioning is idempotent by owner email and school slug.
- A failed provisioning step can be retried without duplicate branches, users, or templates.
- Newly provisioned tenants land in a guided onboarding checklist.

## Subscription and Plan Model

DrivingOps needs tenant-level subscription entitlements independent from school invoices/payments.

### Suggested Plans

| Plan | Target Tenant | Entitlements |
|---|---|---|
| Starter | Small school | 1 branch, 3 instructors, 150 active learners, public site, basic automations |
| Growth | Growing school | 3 branches, 15 instructors, 1,000 active learners, advanced scheduling, WhatsApp flows |
| Multi-Branch | Established school | 10 branches, 75 instructors, reporting exports, priority support |
| Platform/Internal | Nextia/admin | Unlimited, platform tools |

### Required Tables

```sql
plans (
  code pk,
  name,
  billing_interval,
  base_price_cents,
  currency,
  active,
  entitlements jsonb
)

tenant_subscriptions (
  id pk, tenant_id,
  plan_code,
  status,
  current_period_start,
  current_period_end,
  trial_end,
  external_customer_id null,
  external_subscription_id null,
  created_at, updated_at
)

tenant_usage_snapshots (
  id pk, tenant_id,
  active_learners,
  instructors,
  branches,
  whatsapp_messages_sent,
  storage_bytes,
  captured_at
)
```

### Entitlement Checks

Entitlements must be enforced in the API and surfaced in the UI.

Examples:

- `learners.create`: fail with `plan_limit_reached` if active learners exceed plan.
- `instructors.manage`: fail if instructor count exceeds plan.
- `public_site.manage`: disabled if plan excludes public site.
- `automations.manage`: limit flows/channels by plan.

Frontend behavior:

- Show a clear upgrade prompt instead of a generic error.
- Settings -> Billing shows plan, usage, trial end, and upgrade CTA.
- Platform admin can override entitlements for a tenant.

## Platform Admin Console

The `PLATFORM_ADMIN` role needs real operational pages.

Pages:

- Tenant list
- Tenant detail
- Provision tenant
- Impersonation/session handoff with audit
- Subscription/plan override
- Tenant health
- Background jobs
- Support tickets
- Audit log search
- Feature flags

Tenant health metrics:

- API error rate;
- active learners;
- upcoming sessions;
- failed automations;
- unpaid DrivingOps subscription status;
- storage usage;
- last login by owner;
- pending support tickets;
- public site publish status.

Endpoint examples:

```text
GET  /api/v1/platform/tenants
GET  /api/v1/platform/tenants/{tenant_id}
GET  /api/v1/platform/tenants/{tenant_id}/health
POST /api/v1/platform/tenants/{tenant_id}/suspend
POST /api/v1/platform/tenants/{tenant_id}/reactivate
POST /api/v1/platform/tenants/{tenant_id}/impersonation-sessions
```

Impersonation must:

- require `platform.tenants.manage`;
- show a visible banner in the frontend;
- write an audit row;
- never reveal passwords or tokens;
- be disabled for production unless explicitly enabled by policy.

## Onboarding Checklist

Every new tenant should see a guided setup path.

Checklist items:

1. Confirm school profile.
2. Add branches.
3. Configure programs and pricing.
4. Add instructors.
5. Configure operating hours and templates.
6. Connect communication channels.
7. Configure cancellation policy.
8. Publish public site.
9. Register first learner.
10. Generate first schedule.

Data model:

```sql
tenant_onboarding_tasks (
  id pk, tenant_id,
  code,
  status,
  completed_at,
  completed_by,
  metadata jsonb
)
```

Acceptance criteria:

- Empty tenants do not show fake operational metrics.
- The dashboard switches from setup mode to operations mode after core tasks are complete.
- Each checklist item deep-links to the relevant settings/workflow.

## Support and Customer Success

DrivingOps needs an internal support model.

```sql
support_tickets (
  id pk,
  tenant_id,
  requester_user_id,
  subject,
  description,
  category,
  priority,
  status,
  assigned_to_user_id null,
  created_at,
  updated_at,
  resolved_at null
)

support_ticket_messages (
  id pk,
  ticket_id,
  author_user_id null,
  author_type,
  body,
  attachments jsonb,
  created_at
)
```

Frontend:

- Settings -> Support for school users.
- Platform admin -> Support queue.
- Include request ID and tenant ID in support ticket metadata.

## Compliance and Privacy

Minimum Canadian SaaS obligations:

- PIPEDA privacy policy coverage.
- Quebec Law 25 readiness for Quebec schools.
- Explicit consent records for WhatsApp/email.
- Data export and deletion workflow.
- Breach-notification process.
- Data residency statement for Canadian Azure regions.
- Retention schedules for learner records, invoices, certificates, and audit logs.

Required capabilities:

```text
GET  /api/v1/tenants/{tenant_id}/data-export
POST /api/v1/tenants/{tenant_id}/data-export
POST /api/v1/tenants/{tenant_id}/privacy/deletion-requests
GET  /api/v1/platform/privacy/deletion-requests
```

Deletion requests should move through:

```text
requested -> verified -> scheduled -> completed -> rejected
```

## Operational SLAs

Dev has no SLA. Staging and production should target:

| Metric | Target |
|---|---|
| API uptime | 99.9% monthly |
| API p95 latency | < 500 ms for common reads |
| Schedule generation p95 | < 3 s for 50 learners / 30-day window |
| Automation delivery enqueue | < 10 s after triggering event |
| Recovery time objective | < 1 hour |
| Recovery point objective | < 5 minutes |

## Release Management

Each production release should include:

- migration plan;
- rollback plan;
- feature flags;
- tenant-impact notes;
- support script/runbook;
- smoke-test result;
- App Insights dashboard link.

Release artifact:

```text
release_notes/
  YYYY-MM-DD-version.md
```

Template:

```markdown
# DrivingOps Release <version>

## Changes
## Migrations
## Feature Flags
## Rollback
## Tenant Impact
## Verification
## Known Issues
```

## Acceptance Criteria

- A platform admin can provision, suspend, reactivate, and inspect a tenant.
- A school owner can complete onboarding without developer help.
- A tenant cannot exceed plan limits without a clear upgrade path.
- A support ticket can be created from the app and traced with request ID.
- Data export can be generated for one tenant without including another tenant's data.
- Every subscription and lifecycle transition writes an audit row.

## Dependencies

- [02-multitenancy-rbac.md](./02-multitenancy-rbac.md) for membership, tenant status, and platform admin permissions.
- [03-authentication.md](./03-authentication.md) for real owner signup and invitations.
- [09-invoices-payments.md](./09-invoices-payments.md) for Stripe integration; SaaS subscription billing should share PSP primitives but remain separate from school learner invoices.
- [15-settings.md](./15-settings.md) for billing, onboarding, support, and tenant profile UI.
- [16-infrastructure.md](./16-infrastructure.md) for production security, backups, monitoring, and custom domains.
