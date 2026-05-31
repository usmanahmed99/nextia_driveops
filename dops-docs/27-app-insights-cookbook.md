# 27 — App Insights query cookbook

> Companion to [26-observability-plan.md](./26-observability-plan.md). Paste-able KQL queries for the questions you'll actually ask.
>
> **Which table each kind of log lands in** (Azure Monitor OpenTelemetry exporter for Python):
> - `log_event(log, "x.y", ...)` at INFO level → **`traces`** with `customDimensions.event = "x.y"`.
> - `log.warning(...)` → **`traces`** (severityLevel 2).
> - `log.exception(...)` / `log.error(...)` → **`exceptions`**.
> - FastAPI request spans → **`requests`** (auto-instrumented).
> - Outbound HTTP + SQLAlchemy spans → **`dependencies`** (auto-instrumented).
>
> Most queries below `union traces, exceptions` to catch a domain event whether it succeeded or threw.

The fields the templates put on **every** record:

| Field | Where it comes from |
| --- | --- |
| `customDimensions.event` | the second argument to `log_event(log, "event.name", …)` |
| `customDimensions.request_id` | `RequestIdMiddleware` (also the `X-Request-ID` response header) |
| `customDimensions.tenant_id` | `TenantMiddleware`, from the `X-Tenant-ID` request header |
| `customDimensions.user_id` | bound by `get_current_identity` / `get_current_session` once the auth dep resolves |
| `customDimensions.logger`, `module`, `func`, `location`, `cls` | structured prefixes from the formatter |

---

## Diagnostics

### Trace a specific request end-to-end
Grab `X-Request-ID` from the browser Network tab → response headers (the SPA also stamps this on every outgoing call, so client and server share it). Then:

```kql
union requests, dependencies, traces, exceptions
| where timestamp > ago(30m)
| where customDimensions.request_id == "<paste-id-here>"
| project timestamp, itemType, name, message=coalesce(message, outerMessage), customDimensions
| order by timestamp asc
```

### All 5xx in the last hour, grouped by route
```kql
exceptions
| where timestamp > ago(1h)
| where customDimensions.event == "request.unhandled_error"
| extend path = tostring(customDimensions.path)
| summarize errors=count(), sample=any(outerMessage) by path
| order by errors desc
```

### Single tenant's activity (everything they triggered today)
```kql
union traces, exceptions, requests
| where timestamp > ago(24h)
| where customDimensions.tenant_id == "<tenant-id>"
| project timestamp, kind=itemType, name=coalesce(name, operation_Name), customDimensions
| order by timestamp desc
```

### Single user's activity
```kql
union traces, exceptions
| where timestamp > ago(24h)
| where customDimensions.user_id == "<user-id>"
| project timestamp, message, customDimensions
| order by timestamp desc
```

### Slow endpoints (p95 > 2s)
```kql
requests
| where timestamp > ago(1h)
| summarize p50=percentile(duration, 50), p95=percentile(duration, 95), count() by name
| where p95 > 2000
| order by p95 desc
```

---

## Auth & onboarding

### Signup funnel last 7 days
```kql
traces
| where timestamp > ago(7d)
| where customDimensions.event in ("identity.created", "school.registered", "invite.created", "invite.accepted")
| summarize count() by tostring(customDimensions.event), bin(timestamp, 1d)
| order by timestamp asc
```

### Schools created today, with owner ids
```kql
traces
| where timestamp > ago(24h) and customDimensions.event == "school.registered"
| project timestamp,
          tenant_id=tostring(customDimensions.tenant_id),
          owner_user_id=tostring(customDimensions.user_id),
          slug=tostring(customDimensions.slug)
| order by timestamp desc
```

### Invite delivery status (sent vs failed)
```kql
union traces, exceptions
| where timestamp > ago(24h)
| where customDimensions.event in ("invite.created", "email.sent", "email.delivery_failed")
| summarize count() by tostring(customDimensions.event), bin(timestamp, 1h)
```

### Demo / token validation failures
```kql
requests
| where timestamp > ago(1h) and resultCode == 401
| project timestamp, name, customDimensions.request_id, url
| order by timestamp desc
```

---

## Domain activity

### Driving-school operational events
```kql
traces
| where timestamp > ago(24h)
| extend event = tostring(customDimensions.event)
| where event startswith "learner."
   or event startswith "session."
   or event startswith "invoice."
   or event startswith "payment."
   or event startswith "certificate."
   or event startswith "automation."
| summarize count() by event, bin(timestamp, 1h)
```

### Audit trail mirror (anything that wrote to audit_logs)
Audit-table writes also surface here:
```kql
traces
| where timestamp > ago(24h)
| where customDimensions has "entity_type"
| project timestamp,
          action=tostring(customDimensions.event),
          tenant_id=tostring(customDimensions.tenant_id),
          actor=tostring(customDimensions.actor_user_id),
          entity_type=tostring(customDimensions.entity_type),
          entity_id=tostring(customDimensions.entity_id)
| order by timestamp desc
```

### Outbox events (downstream consumer view)
```kql
traces
| where timestamp > ago(1h)
| extend event = tostring(customDimensions.event)
| where event startswith "outbox."
| project timestamp, event, tenant_id=tostring(customDimensions.tenant_id),
          entity_type=tostring(customDimensions.entity_type),
          entity_id=tostring(customDimensions.entity_id)
| order by timestamp desc
```

### Consent sweep job health
```kql
union traces, exceptions
| where timestamp > ago(7d)
| where customDimensions.event in ("consent.sweep_completed", "consent.sweep_failed")
| project timestamp, customDimensions
| order by timestamp desc
```

---

## SLOs / health-at-a-glance

### Error rate, last hour
```kql
requests
| where timestamp > ago(1h)
| summarize total=count(), failed=countif(success == false)
| extend error_rate_pct = round(100.0 * failed / total, 2)
```

### Top exception types, last 24h
```kql
exceptions
| where timestamp > ago(24h)
| summarize count() by type, outerMessage
| top 20 by count_ desc
```

### Cold-start health (uvicorn worker restarts)
```kql
traces
| where timestamp > ago(24h)
| where message contains "Uvicorn running" or message contains "Application startup"
| project timestamp, message, cloud_RoleInstance
| order by timestamp desc
```

---

## Alerts to set up in the portal

These mirror the cookbook queries; portal → Application Insights → Alerts → New alert rule. Action group can route to email / Teams / Slack via webhook.

| Severity | Name | Signal | Threshold |
| --- | --- | --- | --- |
| 1 | API 5xx spike | `requests` failed count where `resultCode startswith "5"` | > 5 in 5 minutes |
| 1 | Unhandled exceptions | `customDimensions.event == "request.unhandled_error"` count | > 3 in 5 minutes |
| 2 | Invite email failures | `customDimensions.event == "email.delivery_failed"` count | > 0 in 5 minutes |
| 2 | p95 latency degrade | `requests \| summarize p95=percentile(duration,95)` | > 3000ms over 15 minutes |
| 3 | Consent sweep failed | `customDimensions.event == "consent.sweep_failed"` count | > 0 in 1 hour |

---

## Optional: ship frontend errors to App Insights too

`createLogger` already accepts a pluggable sink. The minimal wiring once `@microsoft/applicationinsights-web` is installed:

```ts
// src/main.tsx (or wherever you boot the SPA)
import { ApplicationInsights } from "@microsoft/applicationinsights-web";
import { setLogSink } from "@/lib/observability/logger";

const cs = import.meta.env.VITE_APPINSIGHTS_CONNECTION_STRING;
if (cs) {
  const ai = new ApplicationInsights({ config: { connectionString: cs } });
  ai.loadAppInsights();
  setLogSink((entry) => {
    const props = { ...entry, scope: entry.scope };
    if (entry.level === "error") ai.trackException({ exception: new Error(entry.message), properties: props });
    else ai.trackEvent({ name: `${entry.scope}.${entry.level}`, properties: props });
  });
}
```

With the correlation-id header the frontend already sends (`X-Request-ID`), a client error and the backend's view of the same request join cleanly:

```kql
union traces, exceptions, customEvents, requests
| where timestamp > ago(15m)
| where customDimensions.request_id == "<id>"
| order by timestamp asc
```

— shows the SPA failure and the API exception side-by-side.
