# 08 — Automations & Communications

> The booking-confirmation flow (`booking.created` event → bilingual confirmation message with service + location + fees) is specced in [23-service-catalog-and-driving-centers.md](./23-service-catalog-and-driving-centers.md). When the worker lands, that doc is its consumer contract.

## Purpose

Run outbound, event-driven flows on WhatsApp + email: onboarding welcome, missing-documents nudges, payment follow-ups, smart rescheduling, waitlist offers, road-test prep reminders, certificate-ready notifications, inactive-learner re-engagement, and the instructor daily briefing. Each flow can require human approval before sending.

## Current State

### Data model — [`dops-api/app/models/automation.py`](../dops-api/app/models/automation.py)

```python
class AutomationFlow:        # type, name, status, channels, success_rate, last_run_at, messages_sent, escalation_count, requires_human_approval
class AutomationRun:         # flow_id, status, recipient_id, channel, started_at, completed_at, error_message
class WhatsAppConversation:  # learner_id, phone, status, messages (JSON list)
```

### API — [`dops-api/app/api/v1/endpoints/automations.py`](../dops-api/app/api/v1/endpoints/automations.py)

| Method | Path | Perm |
| ------ | ---- | ---- |
| GET | `/automations/flows` | `automations.read` |
| GET | `/automations/flows/{id}` | `automations.read` |
| PATCH | `/automations/flows/{id}` | `automations.manage` |
| GET | `/automations/runs` | `automations.read` |
| GET | `/automations/whatsapp/conversations` | `automations.read` |
| GET | `/automations/whatsapp/conversations/{id}` | `automations.read` |
| POST | `/automations/messages/send` | `automations.manage` |

[`automation_service.send_message`](../dops-api/app/services/automation_service.py) creates a row but **never sends anything**. The `TODO` says "integrate Azure Communication Services WhatsApp/email transport here". No worker reads any queue. The Service Bus queues `dops-automation`, `dops-notifications`, `dops-deadletter` are provisioned but unused.

### UX — [`dops-frontend/src/features/automations/AutomationsPage.tsx`](../dops-frontend/src/features/automations/AutomationsPage.tsx)

- Metric cards: flows enabled / messages sent / approvals needed / escalations.
- Per-flow cards with status, success rate, last run, approve toggle.
- Onboarding timeline preview.
- Recent runs list + WhatsApp queue preview + message-template list (templates are i18n strings, not data).

## Gaps For Real Implementation

1. **No transport**. Nothing connects to WhatsApp or to email. Conversations and runs are persisted but never deliver.
2. **No async worker**. `dops-automation-worker` repo is empty. Service Bus queues are not consumed.
3. **No message templates as data**. Templates are hard-coded i18n strings; need a `MessageTemplate` table with placeholders and per-language bodies.
4. **No event ingestion**. Flow triggers (registration, missing doc, late payment, cancellation, etc.) don't exist as published events; the API mutates state but never emits.
5. **No human-approval queue UX**. The flag exists; no inbox.
6. **No webhook ingress** for delivery status / inbound WhatsApp replies.
7. **No opt-in / opt-out / DNC list**. Required by Canadian anti-spam law (CASL) and WhatsApp Business policy.
8. **No localization-aware sending**. The frontend strings are bilingual; the wire payload isn't.
9. **No instructor briefing scheduler**. The flow type exists; nothing schedules its daily run.
10. **No throttling / rate-limiting / dedupe** of identical messages.

## Real Implementation Plan

### A. Stack choice

- **WhatsApp**: WhatsApp Cloud API via Meta directly, or Azure Communication Services WhatsApp connector. Pick one and put the secrets in Key Vault.
- **Email**: Azure Communication Services Email (already in same Azure resource group footprint) or SendGrid.
- **Inbound (replies, delivery receipts)**: HTTP webhooks → API → Service Bus.

### B. New / changed schema

```sql
message_templates (
  id pk, tenant_id null,                -- null = platform default; tenant-id overrides
  code,                                 -- e.g. 'learner_welcome', 'payment_reminder_1'
  language,                             -- 'en' | 'fr'
  channel,                              -- 'whatsapp' | 'email'
  subject null,                         -- email only
  body,                                 -- mustache-style {{learner.first_name}}
  required_variables jsonb,             -- ['learner.first_name','session.starts_at']
  active boolean,
  updated_at, updated_by user_id
)

automation_flows
  ADD COLUMN trigger_event              -- enum: learner_registered | document_missing | invoice_overdue |
                                        --  session_cancelled | session_reschedule_requested |
                                        --  waitlist_slot_open | road_test_in_7_days | certificate_ready |
                                        --  learner_inactive_14_days | instructor_daily_briefing_due
  ADD COLUMN trigger_schedule_cron null,-- only used by *_briefing_due flows
  ADD COLUMN template_code,             -- pairs with message_templates.code
  ADD COLUMN throttle_per_recipient_hours int

automation_runs
  ADD COLUMN tenant_id, idempotency_key, payload jsonb,
  ADD COLUMN attempt int default 1,
  ADD COLUMN next_attempt_at null,
  ADD COLUMN delivery_id null,          -- provider message id
  ADD COLUMN delivery_state,            -- queued | sent | delivered | read | failed | bounced
  ADD COLUMN delivery_state_updated_at null,
  ADD COLUMN approved_by user_id null, approved_at null

human_approval_items (
  id pk, tenant_id, run_id,
  reason,                               -- enum: requires_human_approval | low_confidence | suspicious_content
  status,                               -- pending | approved | rejected
  decided_by null, decided_at null, decision_note null,
  created_at
)

contact_preferences (
  id pk, tenant_id, user_id,            -- person being contacted (learner / parent / instructor)
  whatsapp_opt_in boolean default false, whatsapp_opted_in_at, whatsapp_opted_out_at,
  email_opt_in boolean default true, email_opted_out_at,
  preferred_language
)

inbound_messages (                      -- ingest from webhook
  id pk, tenant_id, conversation_id null,
  channel, sender_address, body,
  received_at, raw jsonb
)
```

### C. Event-driven architecture

```
[ dops-api ] --(emit domain events to dops-events queue)--> [ Service Bus ]
                                                                 │
                                                                 ▼
                                                       [ dops-automation-worker ]
                                                                 │
                            For each event, matching automation_flows:
                            1) compose payload from event + DB lookups
                            2) check throttle / opt-out / dedupe
                            3) if requires_human_approval → human_approval_items, pause
                            4) else → send via provider, persist delivery_id
                            5) on provider webhook (inbound) → update delivery_state
                                                                 │
                                                                 ▼
                                                          [ provider API ]
```

Events to emit (additions to existing endpoints, not standalone API calls):

| Source endpoint | Event |
| --------------- | ----- |
| `POST /learners` | `learner.registered` |
| `POST /documents` with `pending_review` | `document.uploaded` |
| Document approve | `document.approved` |
| Invoice issue + due-date scanner cron | `invoice.overdue` |
| `POST /schedule/sessions/{id}/cancel` | `session.cancelled` |
| `POST /schedule/sessions/{id}/reschedule` | `session.reschedule_requested` |
| Scheduling daily cron | `session.upcoming_24h` |
| `POST /certificates/{id}/approve` | `certificate.issued` |
| Inactivity scanner cron | `learner.inactive_14d` |
| Daily 06:00 local cron per branch | `instructor.briefing_due` |

### D. Worker repo (`dops-automation-worker`)

Layout:

```
dops-automation-worker/
  src/
    worker.py             entry; loop over dops-automation queue
    handlers/
      learner_registered.py
      document_missing.py
      invoice_overdue.py
      session_cancelled.py
      session_upcoming_24h.py
      waitlist_slot_open.py
      certificate_ready.py
      learner_inactive_14d.py
      instructor_briefing.py
    providers/
      whatsapp.py
      email.py
    templating/
      mustache.py
    persistence/
      api_client.py        -- HTTP client back to dops-api for state writes
  pipelines/
    azure-pipelines-worker.yml
  Dockerfile
  pyproject.toml
```

The worker writes back to the API only through stable endpoints (no shared DB access).

### E. New API endpoints

```
POST   /api/v1/automations/runs                       automations.manage   internal: worker reports state
PATCH  /api/v1/automations/runs/{id}/delivery-state   automations.manage   webhook-fed
POST   /api/v1/automations/inbound                    public, signed       provider webhook → inbound_messages
GET    /api/v1/automations/human-approvals?status=    automations.read
POST   /api/v1/automations/human-approvals/{id}/approve   automations.manage
POST   /api/v1/automations/human-approvals/{id}/reject    automations.manage  body: { note }
GET    /api/v1/message-templates                      automations.manage
PUT    /api/v1/message-templates/{id}                 automations.manage
POST   /api/v1/contact-preferences                    learners.update      opt-in/out edit
```

### F. UX additions

- Automations page → keep current cards but feed every metric from real `automation_runs` aggregates over the current 24h window.
- New tab **Approvals** — paginated table of `human_approval_items` with body preview and Approve/Reject; required role = `automations.manage`.
- New tab **Templates** — list + editor (per language, per channel). Sandbox preview with sample data.
- Learner detail page WhatsApp panel — backed by real conversation rows; reply inputs send via API which forwards to the worker.
- Instructor portal banner — pulls the day's briefing instead of static copy.

### G. Compliance

- WhatsApp Business: pre-approved template messages required for outbound to unsubscribed contacts; conversational messages only inside 24h customer window. Store `whatsapp_session_window_until` per learner.
- CASL: store explicit opt-in source + timestamp + IP in `contact_preferences`; all messages include unsubscribe (email) / `STOP` handling (WhatsApp).
- Rate-limit per recipient per flow per `throttle_per_recipient_hours`.

## Acceptance Criteria

- A new learner registration triggers exactly one outbound WhatsApp welcome (if opt-in) or email welcome, with delivery state tracked.
- A document upload pending review pauses the missing-document flow; document approval resolves it.
- Cancellation of a confirmed session fires the smart-rescheduling flow which either offers a replacement to the same learner or to a waitlist learner, recorded in `replacement_offers`.
- Human-in-the-loop approvals appear in the UI within < 5 s of being created.
- An inbound WhatsApp reply lands on the learner conversation panel within < 10 s of the provider webhook.
- Opt-out via `STOP` flips `whatsapp_opt_in=false` and suppresses future outbound until re-opt-in.
- No flow ever sends without a matching `message_templates` row for the recipient's language; an audit alert fires if missing.

## Dependencies

- [02-multitenancy-rbac.md](./02-multitenancy-rbac.md) — opt-in is per user, audited.
- [04-learners.md](./04-learners.md), [06-scheduling.md](./06-scheduling.md), [09-invoices-payments.md](./09-invoices-payments.md), [10-certificates.md](./10-certificates.md), [11-documents.md](./11-documents.md) — event emitters.
- [16-infrastructure.md](./16-infrastructure.md) — worker Container App + provider secrets.
