# 14 — Reports & Operational Dashboard

## Purpose

Give school owners and admins a real-time operating picture today and historic insight over weeks/months: utilization, revenue, cancellations, learner progress, automation health.

## Current State

### Dashboard — [`dops-frontend/src/features/dashboard/DashboardPage.tsx`](../dops-frontend/src/features/dashboard/DashboardPage.tsx)

- 6 metric cards (sessions today, eligible learners, unpaid invoices, certificates pending, conflicts, messages).
- "Today at a glance" session grid.
- Smart recommendations action panel (cards are static copy).
- Blocked work list, recent cancellations, automation health gauge.
- Instructor utilization chart + quick action buttons.

Almost every metric is computed client-side from mocked or paged data, with several hard-coded fall-back numbers (e.g. "Sessions today: 18", "Messages: 12").

### Reports — [`dops-frontend/src/features/reports/ReportsPage.tsx`](../dops-frontend/src/features/reports/ReportsPage.tsx)

Two charts using mock data:

- Revenue chart (mock numbers).
- Instructor utilization chart (derived from instructors list).

### API — [`dops-api/app/api/v1/endpoints/reporting.py`](../dops-api/app/api/v1/endpoints/reporting.py)

Mock-only `dashboard summary` endpoint(s) returning placeholders (verify).

## Gaps For Real Implementation

1. **No real aggregates**. Counts ought to come from SQL on the live tables.
2. **No time-window selection**. Reports need a date-range picker.
3. **No drill-down**. A clicked metric should navigate to the filtered list.
4. **No export** (CSV/PDF).
5. **No per-branch breakdown**.
6. **Recommendation cards are static copy**. Should be derived ("3 conflicts in next 7 days", "5 invoices crossed 14-day overdue").

## Real Implementation Plan

### A. New aggregate endpoints

```
GET /api/v1/reports/dashboard?as_of=YYYY-MM-DD                reports.read
  -> {
       today: { sessions_count, sessions_completed, sessions_no_show, sessions_cancelled },
       week:  { revenue_cents, new_learners, payments_collected_cents, messages_sent },
       alerts: {
         unpaid_invoices_count, unpaid_invoices_balance_cents,
         conflicts_in_next_7_days,
         certificates_pending_count,
         learners_behind_schedule, learners_missing_documents,
         automations_failed_24h,
         human_approvals_pending
       },
       smart_recommendations: [
         { id, kind, title, detail, primary_action }   -- generated server-side
       ]
     }

GET /api/v1/reports/utilization?from=&to=&branch_id=          reports.read
  -> per-instructor utilization with available_minutes / scheduled_minutes / completed_minutes

GET /api/v1/reports/revenue?from=&to=&group_by=month|week|branch|program     reports.read
  -> series of { period, gross_cents, tax_cents, refunds_cents, net_cents }

GET /api/v1/reports/cancellations?from=&to=&group_by=reason|branch|instructor reports.read

GET /api/v1/reports/learner-progress?from=&to=                reports.read
  -> distribution of progress.overall_percent buckets, average days-to-certificate

GET /api/v1/reports/automation-health?from=&to=               reports.read
  -> per-flow success_rate, p50/p95 latency, failed_runs
```

All endpoints use materialised tables or properly indexed queries. Heavy aggregates (revenue, utilization) are computed by a nightly job into `report_snapshots` and served from there for ranges > 7d; live for shorter windows.

### B. UX changes

- Dashboard metric cards bound 1:1 to the `dashboard` endpoint.
- "Smart recommendations" panel renders the server-provided list.
- Reports page gets a date-range picker, branch picker, and per-section drill-downs:
  - Revenue → invoices table filtered by range.
  - Utilization → instructor profile filtered by range.
  - Cancellations → sessions list filtered by status+reason+range.
  - Automation health → automations runs page filtered by flow.
- Each chart supports CSV download.

### C. Recommendations engine

`reporting_service.recommendations(tenant_id)`:

- learners eligible for next practical (progress thresholds + program prerequisites met),
- conflicts in next 7 days (blocking severity),
- invoices crossed 14d overdue,
- approval queue depth,
- certificates awaiting approval,
- instructors with utilization < 40% over last 14 days (under-used) or > 95% (over-booked).

Returns up to N items, each with a `primary_action.deeplink` to the right page with the right filter.

## Acceptance Criteria

- Dashboard contains no hard-coded numbers; all metrics derived from one endpoint.
- Reports endpoints respond in < 500 ms p95 for ranges up to 90 days.
- CSV export matches the on-screen numbers exactly.
- Recommendation cards always link to the filtered destination page.

## Dependencies

- All operational modules; reporting is downstream of [06-scheduling.md](./06-scheduling.md), [09-invoices-payments.md](./09-invoices-payments.md), [08-automations.md](./08-automations.md), [04-learners.md](./04-learners.md).
