# DrivingOps Docs Agent Guide

This file gives coding agents the working rules for the `dops-docs` repo. It is intentionally independent so an agent can start here without needing the infra repo open.

## Workspace Map

- `dops-docs`: product architecture, roadmap, backlog, implementation workflow, and feature specs. This repo is the source of truth for sequencing.
- `../dops-api`: FastAPI backend, SQLAlchemy models, Alembic migrations, seed data, and tests.
- `../dops-frontend`: React, TypeScript, Vite frontend.
- `../dops-infra`: Azure infrastructure, pipelines, deployment docs, and runbooks.
- `../dops-automation-worker`: future automation worker.

## Documentation Placement

- Product plans, implementation workflow plans, feature specs, roadmap updates, and backlog notes belong in this repo.
- Infrastructure deployment guides, Azure runbooks, pipeline notes, and Bicep documentation belong in `../dops-infra/docs`.
- Backend-specific implementation notes belong in `../dops-api/docs` when they only concern the API.
- Frontend-specific implementation notes belong in `../dops-frontend/docs` when they only concern the UI.
- When a note spans product behavior across repos, keep it here and link to it from repo-specific docs if needed.

## Product Priorities

Work in the natural school setup flow:

1. Auth and tenant foundation.
2. School setup: profile, branding, branches, languages, operating defaults.
3. Team setup: staff, instructors, vehicles, qualifications, availability.
4. Programs, learning paths, services, driving centers, and cancellation rules.
5. Learner registration, holds, documents, payment readiness, and learner workspace.
6. Scheduling, conflict checks, session completion, check-in, attendance, no-show, and lesson notes.
7. Communications, payments, certificates, portals, public site, reports, and hardening.

Keep the app usable at the end of each step. Prefer thin vertical slices that can be verified from the UI and API.

## Source Docs To Check First

- `IMPLEMENTATION_WORKFLOW_PLAN.md`
- `17-build-roadmap.md`
- `19-implementation-backlog.md`
- `20-next-steps.md`
- `15-settings.md`
- `23-service-catalog-and-driving-centers.md`
- `04-learners.md`
- `06-scheduling.md`

## Documentation Practices

- Keep docs actionable and tied to the current implementation order.
- Update implementation workflow notes when product priorities change.
- Keep school setup concepts generic and configurable. Avoid SAAQ-specific names in core product language unless documenting a jurisdiction-specific integration.
- Record performance and UX expectations explicitly: async data fetching, section-level loading states, and avoiding whole-page blocking are product requirements.
- Mention cross-repo contract changes when a feature affects API schemas, frontend contracts, infrastructure, or deployment.
- Never commit secrets, local environment values, generated sensitive outputs, or private customer data.
- Do not push commits automatically. Commit and push only when the user explicitly asks for it.

## Verification

Docs changes usually need review rather than automated tests. Check links, filenames, and whether the note belongs in this repo or in a repo-specific docs folder. When docs are paired with code changes, run the relevant checks in the affected code repo.

## Current Product Rule

Login has been manually verified. Next work should keep making school setup dependable before deeper operational automation: school profile, branding, branches, team, services, programs, learning paths, and learner setup come before scheduling automation.
