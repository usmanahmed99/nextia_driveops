# 27 — WhatsApp Cloud API: Connection Guide

> **Status**: code shipped, credentials pending. The transport
> (`dops-api/app/services/whatsapp_service.py`), the notification wiring, the
> config flag, and the Meta webhook handshake are all implemented. WhatsApp
> notification rows stay `config_pending` until you complete the steps below.
> See also [08-automations.md](./08-automations.md).

## How it behaves today

`NotificationService` writes a `notifications` row per recipient per enabled
channel (cancellations, reschedules, auto-cancels, contingency reminders). For
WhatsApp the row is delivered **only when both** are true:

1. The tenant has **Communication → WhatsApp enabled** and the provider flag
   `whatsappProviderConfigured = true` in tenant settings, **and**
2. The deployment has Cloud API credentials (`WHATSAPP_ACCESS_TOKEN` +
   `WHATSAPP_PHONE_NUMBER_ID`).

Otherwise the row is saved as `config_pending` and the Automations page shows the
"WhatsApp not configured" banner. When live, a successful Cloud API call marks the
row `sent`; a transport error marks it `failed` (it never breaks the booking flow).

## Step 1 — Meta / Facebook prerequisites

1. Create a **Meta Business account**: <https://business.facebook.com>.
2. Go to **Meta for Developers** (<https://developers.facebook.com>) → **My Apps →
   Create App → Business**.
3. In the app, add the **WhatsApp** product. This provisions a free **test phone
   number** you can use immediately for development.
4. Note these from **WhatsApp → API Setup**:
   - **Phone number ID** → `WHATSAPP_PHONE_NUMBER_ID`
   - **WhatsApp Business Account (WABA) ID** (needed for templates)
   - A **temporary access token** (24h) for first tests → `WHATSAPP_ACCESS_TOKEN`

## Step 2 — A permanent access token (for deployment)

The 24h token is dev-only. For production, create a **System User** token:

1. **Business Settings → Users → System users → Add** (role: Admin).
2. **Add assets** → assign your app with full control.
3. **Generate new token** → select the app → scopes **`whatsapp_business_messaging`**
   and **`whatsapp_business_management`** → set **no expiry**.
4. Store that token as `WHATSAPP_ACCESS_TOKEN`.

## Step 3 — Verify a sender phone number (production)

The test number can only message a short allow-list. To message any learner:

1. **WhatsApp → API Setup → Add phone number**, complete business verification.
2. Verify the number (SMS/voice), set the display name.
3. Use that number's **Phone number ID** as `WHATSAPP_PHONE_NUMBER_ID`.

## Step 4 — Message templates (required for proactive sends)

WhatsApp policy: outbound messages to a user **outside** a 24-hour customer-service
window must use a **pre-approved template**. Free-form text only works inside an
open conversation (e.g. a reply within 24h of the user messaging you).

1. **WhatsApp Manager → Message templates → Create template** (category *Utility*
   for transactional notices like cancellations).
2. Author bilingual versions (en + fr) with placeholders, e.g.
   `Your {{1}} on {{2}} was cancelled.`
3. Submit for approval (usually minutes to a few hours).
4. To send a template, the code path is `WhatsAppService.send_template(...)`.
   `send_text` (the current default in `NotificationService._deliver`) is fine for
   testing and for replies inside the 24h window; swap to `send_template` for
   proactive notifications once your templates are approved.

## Step 5 — Webhook (inbound + delivery status)

Already implemented:

- **Verification (GET):** `GET /api/v1/webhooks/whatsapp` echoes `hub.challenge`
  when `hub.verify_token` matches `WHATSAPP_VERIFY_TOKEN`.
- **Events (POST):** `POST /api/v1/webhooks/whatsapp` accepts inbound messages and
  delivery/read receipts (logged today; threading is future work per
  [08-automations.md](./08-automations.md)).

Wire it in Meta:

1. **WhatsApp → Configuration → Webhook → Edit**.
2. Callback URL: `https://<your-api-host>/api/v1/webhooks/whatsapp`.
3. Verify token: a random string you choose; set the **same** value as
   `WHATSAPP_VERIFY_TOKEN`.
4. Subscribe to the **`messages`** field.

## Step 6 — Environment variables

The API reads these (see `dops-api/app/core/config.py`):

| Var | Required | Notes |
| --- | --- | --- |
| `WHATSAPP_ACCESS_TOKEN` | yes | System-user token (no expiry) for prod |
| `WHATSAPP_PHONE_NUMBER_ID` | yes | From WhatsApp API Setup |
| `WHATSAPP_VERIFY_TOKEN` | for webhook | Random string; matches Meta config |
| `WHATSAPP_API_VERSION` | no | Defaults to `v21.0` |

**Local dev (`dops-api/.env`):**
```dotenv
WHATSAPP_ACCESS_TOKEN=EAAG...
WHATSAPP_PHONE_NUMBER_ID=123456789012345
WHATSAPP_VERIFY_TOKEN=choose-a-random-string
```

## Step 7 — Per-tenant opt-in

Credentials alone don't send. In the app, as a school owner/admin:

1. **Settings → Communication** → enable **WhatsApp**.
2. Set the provider-configured flag (`whatsappProviderConfigured`) in tenant
   settings to `true` once the school has confirmed its WhatsApp number / consent.

Both must be on for delivery; `GET /api/v1/notifications/config` returns
`whatsappConfigured: true` only when the tenant flag **and** the deployment
credentials are both present.

## Step 8 — Deploy config (Azure)

**The Bicep is already wired.** The Key Vault secrets (`DOPS-WHATSAPP-ACCESS-TOKEN`,
`DOPS-WHATSAPP-VERIFY-TOKEN`) are created on every deploy (empty by default) and
bound to the API only when populated — so a deploy is safe with nothing set, and
WhatsApp stays config-pending until you supply values.

To turn it on, set these **pipeline variables** in the `dops-dev` (secrets) variable
group — no Bicep edits needed:

| Variable | Secret? | Maps to |
| --- | --- | --- |
| `DOPS_WHATSAPP_ACCESS_TOKEN` | yes | `WHATSAPP_ACCESS_TOKEN` |
| `DOPS_WHATSAPP_VERIFY_TOKEN` | yes | `WHATSAPP_VERIFY_TOKEN` |
| `DOPS_WHATSAPP_PHONE_NUMBER_ID` | no | `WHATSAPP_PHONE_NUMBER_ID` |

`WHATSAPP_API_VERSION` defaults to `v21.0` in the API config; override via env only
if needed. Then redeploy infra.

## Step 9 — Verify end to end

1. `GET /api/v1/notifications/config` → `whatsappConfigured: true`.
2. As a learner with a WhatsApp number, have an admin cancel one of their sessions.
3. The notification row is `sent` (not `config_pending`); a WhatsApp message arrives
   (within the 24h window using `send_text`, or via an approved template otherwise).
4. Reply from the phone → the inbound `POST /webhooks/whatsapp` is logged.

## Compliance reminders

- **CASL / WhatsApp opt-in:** keep explicit consent before messaging. The tenant
  `whatsappOptInRequired` flag and the learner `notification_contact.receives_notifications`
  flag already gate who gets messaged.
- **24h window + templates:** proactive messages need approved templates; honour the
  customer-service window for free-form replies.
- **Opt-out (`STOP`):** handle inbound `STOP` to suppress future outbound — wire this
  into the inbound webhook handler when conversation threading lands.
```
