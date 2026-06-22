# 04 — Legal & Confidentiality Content

The site ships with full, plain-language policy pages under `/legal/*`. They are
written to reflect the product's actual posture (multi-tenant SaaS, Canadian
hosting, RBAC, encryption) and Canadian obligations described in
`dops-docs/18-saas-operations.md` (PIPEDA, Quebec Law 25, consent records, export
& deletion, breach notification, retention).

> **Important:** These pages are drafted for transparency and as a strong starting
> point. **Have them reviewed by qualified legal counsel before launch.** Each page
> carries a visible "not legal advice" note.

## Pages and what each covers

| Page | Route | Covers |
| ---- | ----- | ------ |
| **Legal centre** | `/legal` | Plain-language index and pre-launch legal-review notice |
| **Privacy policy** | `/legal/privacy` | Website/product roles, data categories, purposes, consent/CASL, minors, AI/automated decisions, transfers, retention, access/correction/portability, incidents, privacy officer |
| **Terms of service** | `/legal/terms` | Contract precedence, preview status, accounts, plans/trials/billing, customer responsibilities, data ownership, confidentiality, third parties/AI, suspension/termination, export/deletion, liability, governing law |
| **Acceptable use** | `/legal/acceptable-use` | Security abuse, prohibited content, data minimization, messaging/CASL, responsible AI, platform limits, enforcement |
| **Security & data protection** | `/legal/security` | Residency, encryption, RBAC & tenant isolation, identity, backups/resilience, monitoring, auditability, payments, responsible disclosure |
| **Cookie policy** | `/legal/cookies` | Cookie categories (necessary/preferences/analytics), no ad cookies, management, consent |
| **Accessibility statement** | `/legal/accessibility` | WCAG 2.2 AA design target, AODA legal baseline, current status, known limits, feedback and accessible formats |
| **Data processing & confidentiality (DPA)** | `/legal/dpa` | Processor role, **confidentiality statement**, security measures, sub-processors, data-subject requests, international transfers, breach notification, return & deletion, how to request the signed DPA |
| **Sub-processor list** | `/legal/subprocessors` | Provider, purpose, possible data, processing location, optional/core status, and change notice |

## Confidentiality statements (where they live)

The user specifically asked for confidentiality statements. They appear:

- **DPA page** — a dedicated "Confidentiality statement" section: all customer
  data treated as strictly confidential, access limited, never sold, not used for
  advertising or third-party model training.
- **Terms** — a "Confidentiality" clause (mutual) plus "Customer data & ownership".
- **Security** — the controls that operationalize confidentiality (tenant
  isolation, RBAC, least privilege, audit).

## Before launch — legal checklist

- [ ] Legal review of all six pages and the consent copy on `/demo` & `/contact`.
- [ ] Confirm the registered legal entity name (currently "Nextia AI"), registered
      office, and official notice address.
- [ ] Confirm governing province/jurisdiction in the order form or Terms.
- [ ] Verify the public sub-processor list against production configuration,
      including Cloudflare, Azure, payment, messaging, geocoding, email, identity,
      support, and AI providers.
- [ ] Decide analytics tooling → reconcile with the cookie policy (add a consent
      banner if any non-essential cookies are used).
- [ ] Set real support/sales/privacy email addresses in `src/config/site.ts`.
- [ ] Confirm the published **Privacy Officer** title/contact and internal
      delegation.
- [ ] Verify all security statements against production; do not claim SOC 2,
      ISO 27001, penetration testing, SLA, or disaster-recovery results until true.
- [ ] Align the marketing policies with the shorter in-app `/privacy` and `/terms`
      documents and bump the API `current_terms_version` when approved.
- [ ] Prepare the signed DPA/order form; the website DPA page is a summary only.
</content>
