# 12 — Public Site Builder

## Purpose

Every school gets a bilingual marketing site at `{slug}.drivingops.ca`. Owners edit hero copy, palette, sections, programs displayed, contact info, and resources. Public visitors can read the site, view programs, and start a booking (which becomes a learner registration lead).

## Current State

### Data model — [`dops-api/app/models/public_site.py`](../dops-api/app/models/public_site.py)

```python
class PublicSitePage:
    slug, title, language, sections (JSON), theme (JSON), subdomain, publish_status, published_at
```

`Tenant.public_site` also holds top-level metadata (`subdomain`, `publishStatus`, `publishedAt`).

### API

`public_site.py` endpoints (verify): load/update/publish. The frontend reads via `usePublicSite()`.

### UX

- [`PublicSiteBuilderPage.tsx`](../dops-frontend/src/features/public-site/PublicSiteBuilderPage.tsx) — branding card (logo placeholder, three palette swatches, hero, phone, status badge, subdomain badge), pages list, mobile preview, desktop preview.
- [`PublicSchoolDemoPage.tsx`](../dops-frontend/src/features/public-site/PublicSchoolDemoPage.tsx) — a one-page public demo for the seeded school.

## Gaps For Real Implementation

1. **No actual rendering at the subdomain**. Today the editor previews look but no public HTML exists at `{slug}.drivingops.ca`.
2. **Sections are unconstrained JSON**. Real impl needs typed section variants (hero, features, programs, instructors, testimonials, FAQ, contact, gallery, cta) with validation.
3. **No image hosting**. Logo / hero images are placeholders.
4. **No multi-language editing**. The model has `language` per page but the editor exposes only one set of fields.
5. **No SEO** (meta tags, sitemap, robots, OG image).
6. **No analytics**.
7. **Booking CTA does not exist as a lead model**. A submission has nowhere to go.
8. **No draft / publish history**.

## Real Implementation Plan

### A. Schema additions

```sql
public_sites (
  id pk, tenant_id, subdomain unique,
  custom_domain null, custom_domain_verified_at null,
  default_language,                       -- 'en' | 'fr'
  primary_color, secondary_color, accent_color,
  logo_blob_path null, favicon_blob_path null, og_image_blob_path null,
  google_analytics_id null,
  contact_email, contact_phone, whatsapp_number,
  publish_status,                         -- draft | published
  last_published_at null,
  updated_at, updated_by user_id
)

public_site_pages
  ADD COLUMN page_type,                   -- enum: home | programs | instructors | resources | contact | custom
  ADD COLUMN seo_meta jsonb,
  ADD COLUMN slug unique-within-tenant

public_site_sections (
  id pk, tenant_id, page_id, ordinal,
  variant,                                -- enum: hero | features | programs | instructors | testimonials | faq | contact | gallery | cta
  data jsonb,                             -- variant-specific; validated by pydantic schemas
  language,                               -- 'en' | 'fr'
  enabled boolean
)

public_site_assets (
  id pk, tenant_id,
  kind,                                   -- logo | hero | gallery | other
  blob_path, mime_type, width, height, size_bytes,
  alt_text, alt_text_fr,
  uploaded_at, uploaded_by user_id
)

public_site_revisions (
  id pk, tenant_id, snapshot_at, snapshot jsonb, published_by user_id
)

booking_leads (
  id pk, tenant_id, source,               -- 'public_site_form' | 'whatsapp_link' | 'phone'
  full_name, email, phone, preferred_language,
  program_interest, branch_id null,
  message,
  status,                                 -- new | contacted | converted_to_learner | dismissed
  converted_learner_id null,
  created_at, contacted_by user_id null, contacted_at null
)
```

### B. Rendering strategy

Two options; recommend **option 1**:

1. **Frontend public-site app** served by `dops-frontend` at the wildcard subdomain. SSR with Vite-SSR or migrate to Next.js for SEO. Reads `GET /api/v1/public/{slug}` (no auth) and renders.
2. **Statically pre-built** to Storage `$web` per publish — requires a build job per tenant; more complex CI.

Going with (1) reuses the same React stack with a single, public-facing route group. Add a `Host` middleware that maps `Host` → tenant slug, plus a `/api/v1/public/{slug}` endpoint that returns the published site DTO.

### C. New endpoints

```
GET /api/v1/public/{slug}                     public      -> published site + pages + sections
POST /api/v1/public/{slug}/leads              public, captcha-guarded   creates booking_leads
GET /api/v1/admin/public-site                 public_site.manage
PUT /api/v1/admin/public-site                 public_site.manage
GET /api/v1/admin/public-site/pages
PUT /api/v1/admin/public-site/pages/{id}/sections
POST /api/v1/admin/public-site/publish        public_site.manage    snapshots a public_site_revisions row
POST /api/v1/admin/public-site/assets/upload-link             returns SAS for image upload
```

### D. UX changes

- Builder gets a left-pane structured editor: language tabs (EN/FR), per-section forms with typed fields, drag-to-reorder.
- Right pane keeps the device-frame preview, now fed live from the editor's state.
- Top action: **Publish** snapshots a revision and flips `publish_status`. Banner: "Published 2 minutes ago by Claire".
- New nav item **Leads** (under Learners or Public website): list of `booking_leads`, status transitions, "Convert to learner" → opens the registration wizard pre-filled.

### E. SEO / analytics

- Per-page `seo_meta`: title, description, OG image (asset reference), canonical.
- Robots + sitemap generated from the published revision.
- Optional Google Analytics 4 id stored on `public_sites`; injected by the public app.

### F. Custom domains

- `custom_domain` per tenant; verification via a TXT/CNAME record.
- TLS via Container Apps custom domain feature; document operator runbook.

## Acceptance Criteria

- Editing a section in EN does not change FR copy.
- Publishing snapshots a revision and flips the public app to the new content within < 60 s.
- Booking lead submitted from the public site lands in the Leads queue and can be converted to a learner.
- The public app renders SEO meta + OG image + sitemap; Lighthouse SEO score ≥ 95.
- Subdomain routing works locally (mkcert) and in prod (wildcard cert).

## Dependencies

- [04-learners.md](./04-learners.md) — leads convert to learners.
- [08-automations.md](./08-automations.md) — new-lead notification to school.
- [16-infrastructure.md](./16-infrastructure.md) — wildcard cert + custom-domain hookups.
