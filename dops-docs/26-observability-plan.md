# 26 — Observability rollout plan

> **Goal**: turn the App Insights instance from "you'll find something there if you grep hard" into "every important thing the app does is a queryable, alertable, traceable event". The reusable logging templates already exist (Phase 0 of the auth migration); this doc plans how to **adopt** them everywhere and what to build on top so issues surface fast in App Insights — without chasing logs across Container Apps log streams.

## What's already in place

| Layer | What | Where |
| --- | --- | --- |
| Backend factory | `get_logger(__name__, cls=...)` returns a logger with stable prefixes (module / func / location) | [app/core/logging.py](../dops-api/app/core/logging.py) |
| Backend events | `log_event(log, "event.name", **fields)` emits a structured event with the `event` custom dimension | [app/core/logging.py](../dops-api/app/core/logging.py) |
| Request context | `request_id`, `tenant_id`, `user_id` stamped automatically from contextvars onto every record | [app/core/log_context.py](../dops-api/app/core/log_context.py) |
| Output format | Single-line JSON in deployed envs; human text locally | same |
| Global error handler | `log.exception("unhandled error", ...)` on every 5xx | [app/core/errors.py](../dops-api/app/core/errors.py) |
| App Insights export | `configure_azure_monitor` auto-runs when `APPLICATIONINSIGHTS_CONNECTION_STRING` is set | [app/core/telemetry.py](../dops-api/app/core/telemetry.py) |
| Frontend factory | `createLogger(scope)` with pluggable sink | [src/lib/observability/logger.ts](../dops-frontend/src/lib/observability/logger.ts) |
| Sample adoption | `request.unhandled_error`, `identity.created`, `school.registered`, `invite.created`, `invite.accepted` | services + errors |

The recent `UndefinedColumn: audit_logs.before_json` 500 **did** flow through the global handler: it landed in App Insights with `event=request.unhandled_error`, the full traceback, the request id, and the route. The foundation works. The gap is **adoption breadth + query/alert tooling**, not the templates.

## What's missing (gap analysis)

1. **Service-layer adoption is patchy.** Only `AuthService` and `TenantService` actively call `log_event`. Other services (learner, scheduling, invoices, certificates, automations, service offerings, driving centers, learning paths, consent) raise into the void — when something fails inside them, you have to read a Python traceback instead of filtering by an event name.
2. **DB transaction failures aren't wrapped with context.** When a commit throws (like today's `audit_logs.before_json`), the traceback tells you *what* but not *which payload* or *which step* triggered it. A focused `log.exception("X failed", extra={"action": "...", "tenant_id": ..., "payload_id": ...})` per service method makes diagnosis instant.
3. **Frontend logger is barely used.** `createLogger` is wired in `api/http.ts` and the auth pages, but every other page still has bare `console.error` calls. Client errors don't reach App Insights.
4. **No SPA ↔ backend correlation id.** The browser sees `X-Request-ID` on responses but doesn't propagate it back. A user-reported issue ("it didn't work at 3:42") has to be cross-referenced manually.
5. **No App Insights query cookbook.** Every diagnosis is freeform KQL. The same five queries are typed by hand each time.
6. **No alerts.** A 500 spike in prod is only noticed when a user complains.
7. **No dashboard.** Health is judged by gut.

## The plan, in priority order

### Phase A — Service-layer adoption (highest leverage)

Pattern every service follows:

```python
from app.core.logging import get_logger, log_event

log = get_logger(__name__, cls="LearnerService")

class LearnerService:
    def register(self, tenant_id, payload, actor_user_id, request_meta):
        try:
            learner = self._build(tenant_id, payload)
            self.db.add(learner)
            self.db.commit()
            log_event(log, "learner.registered",
                      tenant_id=tenant_id, learner_id=learner.id,
                      actor_user_id=actor_user_id, program=payload.program_id)
            return learner
        except Exception:
            log.exception(
                "learner.register failed",
                extra={"event": "learner.register_failed",
                       "tenant_id": tenant_id, "actor_user_id": actor_user_id},
            )
            raise
```

Two rules:
- **Success → `log_event(log, "<domain>.<verb>", …)`** with the entity IDs only (no PII).
- **Failure → `log.exception("X failed", extra={"event": "...failed", …})`** so the App Insights `customDimensions.event` field gives you the failure shape.

Concrete event catalogue (per service):

| Service | Success events | Failure events |
| --- | --- | --- |
| `learner_service` | `learner.registered`, `learner.updated`, `learner.withdrawn`, `learner.archived`, `learner.hold_created`, `learner.hold_resolved`, `learner.consent_granted` | corresponding `.failed` |
| `instructor_service` | `instructor.created`, `instructor.updated` | `.failed` |
| `schedule_service` | `session.created`, `session.rescheduled`, `session.cancelled`, `schedule.generated`, `schedule.preview_from_path` | `.failed` |
| `invoice_service` | `invoice.created`, `payment.recorded` | `.failed` |
| `certificate_service` | `certificate.approved` | `.failed` |
| `automation_service` | `automation.flow_updated`, `automation.message_sent` | `.failed` |
| `service_offering_service` | `service_offering.created`, `service_offering.updated`, `service_offering.deactivated` | `.failed` |
| `driving_center_service` | `driving_center.created`, `driving_center.updated`, `driving_center.deactivated` | `.failed` |
| `learning_path_service` | `learning_path.created`, `learning_path.updated`, `learning_path.deleted`, `learning_path.seeded` | `.failed` |
| `consent_service` | `consent.sweep_completed`, `consent.reminder_sent` | `.failed` |
| `tenant_service` | already has invites; **add** `tenant.created` (platform), `tenant.suspended`, `branch.created`, `branch.updated`, `membership.updated`, `membership.removed` | `.failed` |
| `public_site_service` | `public_site.updated`, `public_site.published` | `.failed` |
| `email_service` | `email.sent` (already), add `email.delivery_failed` | already wired |

**Effort**: ~20 minutes per service. Most can be PRs of a few lines each.

### Phase B — DB transaction context

Every `self.db.commit()` is a point where schema drift / constraint violations / FK errors surface. Wrap in a try/except with a structured `log.exception` that includes the parameters relevant to that operation. Today's `audit_logs.before_json` 500 would have produced a one-line query result:

```
exceptions
| where timestamp > ago(1h)
| where customDimensions.event == "service_offering.create_failed"
| project timestamp, outerMessage, customDimensions.tenant_id
```

…instead of the manual traceback dive we just did.

### Phase C — Frontend logger adoption

Replace remaining `console.*` calls (greppable: `rg "console\.(error|warn|log)" src`) with `createLogger("scope/here")`. Per page/feature:

| Area | Scope name | Where |
| --- | --- | --- |
| Learners | `learners/*` | feature dir |
| Scheduling | `schedule` | feature dir |
| Settings | `settings/*` | feature dir |
| Invoices | `invoices` | feature dir |
| Public site | `public-site` | feature dir |

Then wire a real sink so client errors actually land in App Insights:

```ts
import { ApplicationInsights } from "@microsoft/applicationinsights-web";
const ai = new ApplicationInsights({ config: { connectionString: import.meta.env.VITE_APPINSIGHTS_CONNECTION_STRING } });
ai.loadAppInsights();
setLogSink((entry) => {
  if (entry.level === "error") ai.trackException({ exception: new Error(entry.message), properties: entry });
  else ai.trackEvent({ name: entry.scope, properties: entry });
});
```

Gate this with a config flag so local dev doesn't ship telemetry by default.

### Phase D — End-to-end correlation id

Frontend generates an `X-Correlation-ID` per request (UUID v4, stored in sessionStorage for the page lifetime), sends it on every fetch. Backend's `RequestIdMiddleware` uses the incoming header if present (it already does) so it's the same id everywhere. App Insights queries can then join client `customEvents` and server `requests`/`exceptions` by that id — answering "what did the user see when this 500 happened" in one query.

Adds one line to `http.ts`:
```ts
if (!headers.has("X-Request-ID")) headers.set("X-Request-ID", crypto.randomUUID());
```

### Phase E — App Insights query cookbook

Save these to `dops-docs/27-app-insights-queries.md` (or a workbook in the portal). The KQL you'll type most often:

**Find a specific failing request:**
```kql
union requests, exceptions, traces
| where timestamp > ago(15m)
| where customDimensions.request_id == "<paste-from-browser>"
| order by timestamp asc
```

**5xx in last hour, by route, with sample tracebacks:**
```kql
exceptions
| where timestamp > ago(1h)
| where customDimensions.event == "request.unhandled_error"
| summarize count(), any(outerMessage) by tostring(customDimensions.path)
| order by count_ desc
```

**Tenant-scoped activity:**
```kql
traces
| where timestamp > ago(24h)
| where customDimensions.tenant_id == "<tenant-id>"
| order by timestamp desc
```

**Auth funnel (sign-up → school creation):**
```kql
customEvents
| where timestamp > ago(24h)
| where name in ("identity.created", "school.registered", "invite.created", "invite.accepted")
| summarize count() by name, bin(timestamp, 1h)
```

**Slow endpoints:**
```kql
requests
| where timestamp > ago(1h) and duration > 2000
| summarize p95=percentile(duration, 95), count() by name
| order by p95 desc
```

**Invite delivery health:**
```kql
union customEvents, exceptions
| where timestamp > ago(24h)
| where customDimensions.event startswith "invite." or customDimensions.event startswith "email."
| project timestamp, eventName=coalesce(name, "exception"), customDimensions
| order by timestamp desc
```

### Phase F — Alerts

Three alerts to set up in Application Insights → Alerts (Azure portal, no code):

1. **5xx spike** — Signal: `requests/failed` count > 5 in 5 minutes. Severity 2. Action group: email/Slack/Teams.
2. **Unhandled exceptions** — Signal: count of `exceptions` where `customDimensions.event == "request.unhandled_error"` > 3 in 5 minutes. Severity 1.
3. **Invite send failures** — Signal: `customEvents` where `name == "email.delivery_failed"` > 0 in 5 minutes. Severity 3.

Optional later: tenant-scoped error rate (>X% of requests failing for a single tenant_id) to catch single-tenant breakage.

### Phase G — Health dashboard (workbook)

Pin a workbook in App Insights showing, refreshed every minute:

- Request volume + error rate, last 24h, by route.
- p50/p95 duration by route.
- Auth funnel daily totals: signups, school creations, invites sent, invites accepted.
- Top exception types in the last hour.
- Tenant activity heatmap (which tenants are doing things right now).

Most of this is checkbox configuration in the portal, ~half a day.

## Sequencing

| Order | Phase | Estimated effort | Why this slot |
| --- | --- | --- | --- |
| 1 | **A** — service-layer adoption | ~1 day spread over PRs | Everything else compounds on this |
| 2 | **B** — DB transaction context | merged with A | Zero marginal cost when you're already in the file |
| 3 | **E** — query cookbook | half day | Lets you actually use the data A produced |
| 4 | **F** — alerts | half day | Catch things before users do |
| 5 | **D** — correlation id | half day | Cheap, big diagnostic win |
| 6 | **C** — frontend adoption + sink | half day | Useful but less critical than backend |
| 7 | **G** — dashboard | half day | Polish; the queries from E cover ad-hoc needs |

Total: ~3-4 days of focused work, fully landing observability you can actually rely on.

## What you do NOT need to build

- A second logging library — the templates are the standard.
- A custom telemetry sink — App Insights + OpenTelemetry already covers it.
- Per-request manual id passing — contextvars + middleware handle it.
- A Slack bot or custom alerting — App Insights alerts can post to Teams/Slack/PagerDuty out of the box.

## Acceptance criteria (so you know it's done)

- [ ] Every service in [app/services/](../dops-api/app/services/) imports `get_logger` and uses `log_event` for at least the create/update/delete paths.
- [ ] Every `self.db.commit()` is inside a try/except that logs the failure with structured context.
- [ ] Frontend `console.error` count is zero outside of test files.
- [ ] All requests sent by the SPA carry `X-Request-ID`.
- [ ] The query cookbook doc exists and at least three queries are runnable verbatim.
- [ ] Three alerts are active in App Insights.
- [ ] A single dashboard workbook is pinned in the portal showing request rate, error rate, and the auth funnel.
