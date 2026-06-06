# 31 — Public Marketing Site: Domains & Operations

> **Status**: the public marketing site is shipped and data-driven. Each school's
> site is rendered by the SPA from a no-auth API read model. This doc covers how
> the site is reached (path, subdomain, custom domain) and the DNS/cert runbook.
> See also [12-public-site.md](./12-public-site.md).

## What's built

| Piece | Where |
| --- | --- |
| Public read model (school, brand, hero/features/FAQ, programs, stats, contact) | `dops-api` `GET /api/v1/public/sites/{slug}` (no auth, published only) |
| Booking-lead capture (+ staff notification) | `POST /api/v1/public/sites/{slug}/leads` |
| Staff Leads queue | `GET/PATCH /api/v1/public-site/leads` → `/public-website/leads` |
| Builder (hero EN/FR, CTA, section toggles, publish, view-live) | `dops-frontend` `PublicSiteBuilderPage` |
| Polished public page (brand-coloured hero, programs, reviews, FAQ, booking form, WhatsApp) | `dops-frontend` `PublicSitePage` |
| Host → slug resolver | `dops-frontend` `lib/publicSite/resolveSlug.ts` |

The page renders entirely from the school's own data + brand palette, so every
school's site is on-brand and bilingual (EN/FR) automatically.

## How a site is reached

There are three layers, in increasing setup cost. Only the first is needed to go
live; the others are progressive enhancements.

### 1. Path (works today, no DNS)

`https://<app-host>/site/<slug>` — e.g. `https://app.driveops.com/site/maple-leaf`.
This is the fallback and the local-dev path (`http://localhost:5173/site/maple-leaf`).
Nothing to configure.

### 2. Subdomain (`maple-leaf.driveops.com`)

The SPA detects a school subdomain from the **hostname** and renders the public
site at `/` (no auth, no admin shell). The resolver (`resolveSlug.ts`):
- maps `maple-leaf.driveops.com` → slug `maple-leaf`;
- treats `app` / `admin` / `www` / `api` / `portal` and the apex as the platform app
  (not a school);
- only single-label subdomains are slugs (`a.b.driveops.com` is not).

**To enable subdomains for the deployed app:**
1. **Wildcard DNS:** add `*.driveops.com` → the frontend Container App (CNAME to the
   app's default hostname, or an A/ALIAS via your DNS provider).
2. **Wildcard TLS:** add `*.driveops.com` as a **custom domain + managed certificate**
   on the frontend Container App (Container Apps supports wildcard custom domains;
   or front it with Azure Front Door / a wildcard cert). Bind the cert.
3. **CORS / API base:** the SPA calls the API at `VITE_API_BASE_URL`; keep that
   pointing at the API host (it already works cross-origin). Add the new public
   origins to `DOPS_ALLOWED_CORS_ORIGINS` if the public page calls the API from the
   subdomain origin (it does — the lead POST). A wildcard origin isn't supported by
   the CORS list, so either list known school origins or front public traffic
   through the same origin as the API.
4. Edit `PLATFORM_APEXES` in `resolveSlug.ts` if you use a different apex.

> The slug must match `tenant.slug`. The school's subdomain is shown in the builder
> as `<slug>.driveops.com`.

### 3. Custom domain (`drive.maple-leaf.ca`) — optional, non-critical

The schema already carries `custom_domain` on the tenant public-site metadata.
Resolution is **server-side**: the API looks a site up by slug; to support a fully
custom domain you map the custom host to the slug. Runbook per school:

1. School adds a **CNAME** from their domain (e.g. `drive.maple-leaf.ca`) to the
   platform's public host.
2. Add the custom domain + **managed certificate** on the frontend Container App and
   bind it (Container Apps free managed certs cover custom domains).
3. Map the host → slug: either store `custom_domain` on the tenant and add a small
   host→slug lookup in the public resolver path, or run the custom domain through
   Front Door with a route that injects the slug. (This last hop is the only part
   not automated; treat it as an ops task per school — it's deliberately out of scope
   for the app code since it's low-volume.)

## Publish / unpublish

- The public read model returns **404 unless `publishStatus == "published"`**, so a
  school's site is invisible until they hit **Publish** in the builder.
- Editing content (hero/CTA/toggles) saves a draft; **Publish** flips it live.

## Leads

- The public **"Request a callback"** form creates a `booking_lead` and drops an
  in-app staff notification ("New website enquiry").
- Staff manage them at **Leads** (`/public-website/leads`): New → Contacted →
  Converted, or Dismiss. (Converting to a full learner record via the wizard is a
  follow-up enhancement.)

## Acceptance / verification

- `GET /api/v1/public/sites/<slug>` returns the published site DTO; an unpublished
  or unknown slug 404s.
- Visiting `/site/<slug>` (or `<slug>.driveops.com` once DNS is set) renders the
  branded page; the language toggle swaps EN/FR.
- Submitting the booking form creates a lead visible under **Leads** and a staff
  notification.
- Lighthouse/SEO hardening (meta tags, OG image, sitemap) remains future work per
  [12-public-site.md](./12-public-site.md).
