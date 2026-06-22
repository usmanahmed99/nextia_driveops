# DrivingOps Marketing Website — Plan & Brief

This folder is the **single source of truth** for the DrivingOps product marketing
website (the top-of-funnel site that sells the DrivingOps SaaS to driving schools).

It is **not** the per-school public site builder that ships inside the product
(`dops-frontend/src/features/public-site`). That builder generates a marketing
page for each *tenant* school. This site markets **DrivingOps itself** to school
owners and is hosted separately on **Cloudflare Pages** using **Astro**.

## Contents

| File | What it covers |
| ---- | -------------- |
| [`00-strategy.md`](./00-strategy.md) | Positioning, audience, value props, messaging pillars, voice & tone |
| [`01-sitemap-and-pages.md`](./01-sitemap-and-pages.md) | Full sitemap, every page, section-by-section content outline |
| [`02-brand-and-design-system.md`](./02-brand-and-design-system.md) | Colours, type, spacing, components, motion, accessibility |
| [`03-asset-and-screenshot-plan.md`](./03-asset-and-screenshot-plan.md) | Every image/asset the site needs + the in-app screenshot shot list |
| [`04-content-legal.md`](./04-content-legal.md) | Privacy, Terms, Security, Cookies, Accessibility, confidentiality statements |
| [`05-tech-and-hosting.md`](./05-tech-and-hosting.md) | Astro setup, Cloudflare Pages deploy, SEO, analytics, forms, performance |
| [`06-handoff-to-design.md`](./06-handoff-to-design.md) | What the design team should produce from the captured screenshots |

## Status

- [x] Plan written
- [x] App screenshots captured (mock seed data — NorthStar Driving School)
- [x] Astro site scaffolded and built (see `../marketing-website`)
- [ ] Design team replaces raw screenshots with polished, framed assets
- [ ] Real copy review / legal review of policy pages
- [ ] Connect form backend + analytics + deploy to Cloudflare

## Quick facts about the product (for copywriters)

- **What it is:** Bilingual (EN/FR) multi-tenant SaaS that runs a Canadian driving
  school end to end — learners, instructors, vehicles, scheduling, automations,
  billing, certificates, documents, courses, portals, reports.
- **Demo tenant:** *NorthStar Driving School* (Montreal + Laval, QC). All
  screenshots use this seeded data.
- **Who buys it:** Driving-school owners and administrators in Canada.
- **Plans:** Starter, Growth, Multi-Branch (see `01-sitemap-and-pages.md` → Pricing).
- **Differentiators:** one workspace for the whole school, real schedule
  generation, WhatsApp + email automations, learner & instructor portals,
  Canadian compliance (PIPEDA / Quebec Law 25), bilingual by default.
</content>
</invoke>
