# 15 — Settings

## Purpose

Tenant-level configuration. Today the Settings page is a grid of 11 placeholder cards. Real implementation turns each card into an editable section backed by API endpoints.

## Current State

[`dops-frontend/src/features/settings/SettingsPage.tsx`](../dops-frontend/src/features/settings/SettingsPage.tsx) renders cards for:

```
profile, branding, branches, programs, sessions, policy,
communication, whatsapp, roles, language, billing
```

None are functional. The seed sets reasonable defaults on `tenant.settings` and `tenant.profile` but nothing reads/writes them through this page.

## Sections And Where They Are Backed In Real Impl

| Section | Backing model(s) | Cross-ref doc |
| ------- | --------------- | ------------- |
| **Profile** | `tenants.profile` (legal name, address, contact) | this doc |
| **Branding** | `public_sites` colors + logo | [12-public-site.md](./12-public-site.md) |
| **Branches & Locations** | `branches` | [02-multitenancy-rbac.md](./02-multitenancy-rbac.md) |
| **Programs** | `programs` + `program_items` | [07-school-schedule-templates.md](./07-school-schedule-templates.md) |
| **Schedule Templates / Session types** | `schedule_templates`, `class_periods`, `operating_hours`, `closures` | [07-school-schedule-templates.md](./07-school-schedule-templates.md) |
| **Cancellation policy** | `cancellation_policies` | [07-school-schedule-templates.md](./07-school-schedule-templates.md), [06-scheduling.md](./06-scheduling.md) |
| **Communication preferences** | `tenants.profile.communication` + `message_templates` | [08-automations.md](./08-automations.md) |
| **WhatsApp opt-in settings** | per-tenant defaults + provider numbers | [08-automations.md](./08-automations.md) |
| **Staff roles & permissions** | `tenant_memberships`, `invitations` | [02-multitenancy-rbac.md](./02-multitenancy-rbac.md) |
| **Language defaults** | `tenants.profile.communication.default_language`, `supported_languages` | this doc |
| **Billing placeholder** | platform plan, current usage | platform-level (separate doc TBD) |
| **Vehicles** *(new)* | `vehicles`, `instructor_vehicles` | [05-instructors.md](./05-instructors.md) |
| **Tax** *(new)* | per-tenant GST/QST overrides | [09-invoices-payments.md](./09-invoices-payments.md) |
| **Audit log** *(new)* | `audit_log` viewer | [02-multitenancy-rbac.md](./02-multitenancy-rbac.md) |

## Real Implementation Plan

### A. Endpoint surface

```
GET    /api/v1/settings/profile                   settings.manage
PUT    /api/v1/settings/profile                   settings.manage
GET    /api/v1/settings/communication             settings.manage
PUT    /api/v1/settings/communication             settings.manage
GET    /api/v1/settings/branding                  settings.manage    proxy to public-site palette/logo
PUT    /api/v1/settings/branding                  settings.manage
GET    /api/v1/settings/branches                  settings.manage
POST   /api/v1/settings/branches                  settings.manage
PATCH  /api/v1/settings/branches/{id}             settings.manage
DELETE /api/v1/settings/branches/{id}             settings.manage    soft delete; reject if active sessions
```

The Programs / Schedule templates / Cancellation policy / Staff / Vehicles / Audit-log endpoints are owned by their respective docs and just **mounted in the Settings page UI** for ergonomics.

### B. UX

Replace the 11 static cards with a left-nav layout:

```
Settings
├ Profile               (this doc)
├ Branches & Locations  (this doc)
├ Branding              (12-public-site.md)
├ Programs              (07-school-schedule-templates.md)
├ Schedule Templates    (07-school-schedule-templates.md)
├ Cancellation Policy   (07-school-schedule-templates.md)
├ Vehicles              (05-instructors.md)
├ Communication         (this doc + 08-automations.md)
├ Message Templates     (08-automations.md)
├ Staff & Permissions   (02-multitenancy-rbac.md)
├ Languages             (this doc)
├ Tax                   (09-invoices-payments.md)
├ Billing & Plan        (platform billing)
└ Audit Log             (02-multitenancy-rbac.md)
```

Each section uses a consistent header (description + Save button) and forms validated by Zod against the same schemas the API uses.

### C. Validation rules

- Profile: legal name + address + contact required; postal code & phone validators.
- Branches: at least one branch; deleting the last branch rejected.
- Languages: at least EN or FR; default_language must be in supported_languages.
- Communication: WhatsApp opt-in default + email default.

## Acceptance Criteria

- Every section saves to the API and reloads from the API.
- Removing required values shows validation messages without losing user input.
- Owner can update the school address and see it reflected on the public site within < 60 s of publish.
- Permission-gated: Owner sees Billing + Staff; School Admin does not see Billing.
- Sections delegated to other docs work end-to-end through this page.

## Dependencies

Cross-cutting across most docs; see the table above.
