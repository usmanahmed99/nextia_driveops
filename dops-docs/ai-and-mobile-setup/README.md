# AI & Mobile — Setup and Run Guides

This subfolder holds the **manual configuration** and **how-to-run** guides for the features that
need steps outside the normal deploy: the AI features (Azure OpenAI), and the mobile app
(`../../dops-mobile`) including push notifications.

Everything here degrades gracefully when unconfigured — the app and API run fine without any of it.
These guides are for turning each piece **on**.

| Guide | What it covers |
| --- | --- |
| [01 - Azure OpenAI setup](01-azure-openai-setup.md) | Provision the model + wire keys/managed identity; enable AI features and quotas |
| [02 - Run the mobile app](02-run-the-mobile-app.md) | Run `dops-mobile` on an emulator and on your own phone via Expo / expo.dev |
| [03 - Push notifications setup](03-push-notifications-setup.md) | EAS project, `expo_push` channel, end-to-end push verification |

Related existing guides: [29 - Stripe payments](../29-stripe-payments-setup.md) (subscription billing
that plans/discounts plug into), [30 - WhatsApp Cloud API](../30-whatsapp-cloud-api-setup.md)
(notification transport alongside push), [25 - Entra External ID](../25-auth-entra-external-id.md)
(the identity provider the mobile app also uses).
