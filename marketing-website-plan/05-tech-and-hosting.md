# 05 — Tech, Hosting & Operations

## Stack

| Concern | Choice | Why |
| ------- | ------ | --- |
| Framework | **Astro 4** (static output) | Ships zero JS by default → fast, great SEO; islands only where needed (menu, forms, scroll reveal) |
| Styling | **Tailwind CSS 3** | Same tokens as the product (`navy/electric/ice`, Inter) |
| Icons | **lucide-astro** | Same icon set as `dops-frontend` (lucide-react) |
| Fonts | **@fontsource-variable/inter** | Self-hosted Inter → no third-party request, privacy-friendly |
| Content | **Astro content collections** (`src/content/blog`) | Type-safe markdown blog |
| Sitemap | **@astrojs/sitemap** (pinned `3.2.1`) | Auto sitemap; 3.2.1 is the last version compatible with Astro 4 — do not bump without moving to Astro 5 |
| Hosting | **Cloudflare Pages** | Cheap/free, global CDN, easy custom domains, Pages Functions for the form |

## Local development

```bash
cd marketing-website
npm install
npm run dev      # http://localhost:4321
npm run build    # static output to dist/
npm run preview  # serve the production build
```

## Project layout

```
marketing-website/
  astro.config.mjs        # site URL, integrations, output: static
  tailwind.config.mjs     # brand tokens
  src/
    config/site.ts        # all content data (nav, features, plans, FAQs)
    layouts/              # BaseLayout, LegalLayout
    components/           # Header, Footer, Button, BrowserFrame, etc.
    lib/icon.ts           # getIcon() helper
    pages/                # routes (index, features, pricing, …, legal/*, blog/*)
    content/blog/         # markdown posts + config.ts schema
    styles/global.css
  public/
    brand/                # logos copied from the product
    screenshots/          # captured product screenshots + CAPTURE-LOG.md
    robots.txt
  functions/api/lead.js   # Cloudflare Pages Function for form posts
```

## Deploy to Cloudflare Pages

1. Push `marketing-website/` to a Git repo (or a subdirectory in the monorepo).
2. In Cloudflare → **Pages → Create project → Connect to Git**.
3. Build settings:
   - **Framework preset:** Astro
   - **Build command:** `npm run build`
   - **Build output directory:** `dist`
   - **Root directory:** `marketing-website` (if in a monorepo)
4. Add a custom domain: `www.drivingops.ca` (and redirect apex `drivingops.ca` → `www`).
5. Set environment variables (optional, for the form): `LEAD_FORWARD_URL`,
   `TURNSTILE_SECRET`.

`functions/api/lead.js` is automatically deployed as a Pages Function at
`/api/lead` — no extra config.

## Forms

Both `/demo` and `/contact` post JSON to `/api/lead`:

- **Spam protection:** a honeypot field (`company_website`) is included now; add
  **Cloudflare Turnstile** before launch for stronger protection.
- **Where leads go (pick one before launch):**
  1. Forward to the DrivingOps API public leads endpoint
     (`POST {API_BASE}/public/sites/{slug}/leads` or a dedicated marketing
     endpoint) — reuses the existing `BookingLead` pipeline.
  2. Send to a CRM/transactional-email provider via webhook.
  3. Store in Cloudflare KV / D1 and email a digest.
- The client JS degrades gracefully: if `/api/lead` isn't wired yet, the user
  still sees a success acknowledgement (so the site is demoable before backend
  hookup). **Remove that fallback's soft wording once the backend is live.**

## SEO

- Per-page `<title>`, meta description, canonical, Open Graph + Twitter cards
  (in `BaseLayout`).
- `SoftwareApplication` JSON-LD on every page.
- `sitemap-index.xml` generated at build; `robots.txt` points to it.
- Update `SITE` in `astro.config.mjs` and `site.url` in `src/config/site.ts` to
  the production domain before launch (currently `https://www.drivingops.ca`).
- **OG images:** currently the logo; design team should produce per-page 1200×630
  composites (see `03-asset-and-screenshot-plan.md`) and set `ogImage` per page.

## Analytics

- Recommend **Cloudflare Web Analytics** (privacy-friendly, no cookie banner
  needed) or a self-hosted Plausible. Add the snippet in `BaseLayout.astro`.
- If a cookie-setting analytics tool is chosen, add a consent banner (the cookie
  policy already anticipates this).

## Performance budget

- Static HTML, system-island JS only (~a few KB gzipped). Images are PNG
  screenshots — **design team should export optimized WebP/AVIF** versions and
  the build can use Astro's `<Image>` for responsive sizes if desired.
- Target Lighthouse: Performance ≥ 95, SEO ≥ 95, Accessibility ≥ 95.

## Domains (relationship to the product)

- `www.drivingops.ca` → this **marketing** site (Cloudflare Pages).
- `app.drivingops.ca` → the **product** SPA (`dops-frontend`).
- `{school}.drivingops.ca` → per-school public sites (handled by the product; see
  `dops-docs/31-public-site-domains.md`).

Keep these separate; the marketing site links to `app.drivingops.ca` for "Sign in".
</content>
