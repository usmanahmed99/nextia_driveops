# 01 — Sitemap & Page-by-Page Content

## Sitemap

```
/                         Home
/features                 Features overview (all modules)
/features/scheduling      (optional deep-dive — stubbed in v1 via anchors)
/pricing                  Plans, comparison, FAQ
/for-schools              Buyer-focused: owners & administrators
/for-learners             How the learner/instructor experience looks
/about                    Company, mission, Canada/bilingual story
/contact                  General contact + offices
/demo                     Book a demo / start trial (primary conversion form)
/blog                     Resource hub (index + 2–3 seed posts)
/blog/[slug]              Article
/legal                    Legal and trust centre
/legal/privacy            Privacy Policy (PIPEDA / Law 25)
/legal/terms              Terms of Service
/legal/acceptable-use     Acceptable Use Policy
/legal/security           Security & data protection statement
/legal/cookies            Cookie Policy
/legal/accessibility      Accessibility statement (AODA / WCAG)
/legal/dpa                Data Processing Addendum (confidentiality)
/legal/subprocessors      Sub-processor list
/404                      Not found
sitemap.xml, robots.txt   Generated
```

### Global navigation (header)

`Features` · `Pricing` · `For Schools` · `Resources ▾ (Blog, About, Contact)` ·
`Sign in` (→ app) · **Book a demo** (button)

### Footer

- **Product:** Features, Pricing, For Schools, For Learners, Security
- **Company:** About, Blog, Contact, Careers (stub)
- **Legal:** Legal centre, Privacy, Terms, Acceptable Use, Cookies,
  Accessibility, DPA, Sub-processors
- **Get started:** Book a demo, Sign in
- Bilingual toggle (EN/FR — EN only wired in v1), social, "Made in Canada 🍁",
  copyright, address.

---

## Home (`/`)

1. **Hero** — Headline: *"The operating system for modern driving schools."*
   Sub: one workspace for scheduling, learners, billing, and automations —
   bilingual, built for Canada. CTAs: Book a demo / See pricing. Hero visual:
   framed **Dashboard** screenshot (`dashboard.png`) in a browser mock.
2. **Logo strip / trust bar** — "Trusted by driving schools across Canada"
   (placeholder logos) + bilingual + PIPEDA badges.
3. **Problem → solution** — 3 short columns: the spreadsheet chaos vs. one
   workspace.
4. **Feature highlights (6 cards)** — Scheduling, Learners, Instructors,
   Automations, Billing, Certificates. Each: icon, title, one line, link.
5. **Showcase 1 — Scheduling** — Alternating image+text. Screenshot:
   `schedule.png`. Copy on conflict-free generation, rescheduling, vehicle
   assignment.
6. **Showcase 2 — Automations** — Screenshot `automations.png`. WhatsApp + email
   reminders, fewer no-shows.
7. **Showcase 3 — Portals** — Screenshots `learner-portal.png` +
   `instructor-portal.png` in phone frames. Mobile-first learner & instructor
   experience.
8. **Outcomes / stats band** — Big numbers (labelled illustrative).
9. **Bilingual + Canada band** — EN/FR, Law 25/PIPEDA, Canadian residency.
10. **Testimonial** — single strong quote (placeholder) with avatar.
11. **Pricing teaser** — 3 plan cards condensed → link to `/pricing`.
12. **Final CTA band** — "See DrivingOps run your school." Book a demo.

## Features (`/features`)

- Intro hero (short).
- Section per module, each with screenshot + bullet list + "what it replaces":
  **Scheduling & sessions**, **Learners**, **Instructors & availability**,
  **Vehicles & license categories**, **Automations & communications**,
  **Billing & payments**, **Certificates**, **Documents**, **Courses &
  assessments**, **Learner & instructor portals**, **Public site builder**,
  **Reports & dashboard**, **Settings & roles (RBAC)**, **AI assistant**.
- Closing CTA.

## Pricing (`/pricing`)

- Headline + billing toggle (monthly/annual — visual only in v1).
- **3 plan cards:** Starter, Growth (most popular), Multi-Branch. Each: price
  (placeholder CAD), tagline, key entitlements, CTA. Custom/Enterprise note for
  large/multi-branch.
- Entitlements drawn from `dops-docs/18-saas-operations.md`:
  - **Starter** — 1 branch, 3 instructors, 150 active learners, public site,
    basic automations.
  - **Growth** — 3 branches, 15 instructors, 1,000 active learners, advanced
    scheduling, WhatsApp flows. *(Most popular)*
  - **Multi-Branch** — 10 branches, 75 instructors, reporting exports, priority
    support.
- **Comparison table** — feature × plan matrix.
- **Pricing FAQ** — billing, trial, migration, cancellation, what counts as an
  "active learner", taxes.
- CTA band.

## For Schools (`/for-schools`)

Buyer-focused narrative:
- Hero: "Run your whole school from one place."
- By role: **Owners** (visibility, growth, branches, payouts), **Administrators**
  (booking, payments, documents), **Instructors** (today at a glance).
- "A day with DrivingOps" timeline.
- Migration & onboarding ("We move your data; go live in days").
- ROI mini-calculator copy (static in v1).
- CTA.

## For Learners (`/for-learners`)

How schools' learners experience it (sells the buyer on learner delight):
- Book lessons, get reminders, pay online, track progress, get certificates.
- Phone-framed portal screenshots.
- Bilingual experience.
- CTA: "Give your learners this experience — book a demo."

## About (`/about`)

Mission, the Canada + bilingual story, the team (placeholder), values
(reliability, privacy, plain software), "Built by Nextia AI". Contact CTA.

## Contact (`/contact`)

- Contact form (name, school, email, phone, message).
- Offices / region, support email, sales email, response-time note.
- Link to Book a demo.

## Demo (`/demo`) — primary conversion

- Two-column: left = value recap + what to expect (15-min call, tailored to your
  school, no obligation); right = **form** (name, school name, role, email, phone,
  province, # instructors, current tools, preferred language, message, consent
  checkbox). `intent` hidden field (demo|trial).
- Trust: privacy line, "we never share your data".

## Blog / Resources (`/blog`)

- Index grid (category chips, featured post).
- Seed posts:
  1. "5 ways driving schools lose money to no-shows (and how to stop it)"
  2. "What Quebec Law 25 means for your driving school's data"
  3. "From spreadsheet to schedule: how to onboard onto DrivingOps in a week"

## Legal (`/legal/*`)

See [`04-content-legal.md`](./04-content-legal.md): Privacy, Terms, Security,
Cookies, Accessibility, DPA/confidentiality.
</content>
