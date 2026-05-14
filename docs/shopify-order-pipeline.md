# Shopify Order Pipeline

Webhook-driven order processing for Shopify storefronts: extracts key fields from each new order, routes high-value orders to an ops Telegram alert, and tags the customer in Klaviyo by segment.

## What it does

1. Receives a `POST` from Shopify when a customer creates an order (`orders/create` webhook).
2. Extracts the fields the downstream workflow actually uses (order ID, customer name + email, total, currency, line item count, storefront).
3. Branches on total price:
   - **High-value (> 200 currency units):** posts a formatted alert to the ops Telegram channel **and** tags the customer in Klaviyo as `vip-prospect`.
   - **Standard:** tags the customer in Klaviyo as `new-customer`.
4. Responds `200 OK` with the order ID so Shopify marks the webhook as delivered.

## Nodes

| # | Node | Type | Purpose |
|---|---|---|---|
| 1 | Shopify orders/create Webhook | `webhook` | Public endpoint, registered as a Shopify webhook |
| 2 | Extract Order Fields | `set` | Pull the fields we need out of `$json.body` |
| 3 | High-Value Order? | `if` | Branch on `total_price > 200` |
| 4 | Alert Ops Channel | `telegram` | Markdown post to ops chat (true branch only) |
| 5 | Klaviyo — Tag VIP | `httpRequest` | `POST /api/profile-import/` with `vip-prospect` segment |
| 6 | Klaviyo — Tag Standard | `httpRequest` | Same endpoint, `new-customer` segment (false branch) |
| 7 | Respond 200 | `respondToWebhook` | Echo the order ID back |

## Required credentials and env vars

| Variable | Where | Notes |
|---|---|---|
| `KLAVIYO_API_KEY` | n8n env vars | Private API key (`pk_…`) with profile-write scope |
| `TELEGRAM_OPS_CHAT_ID` | n8n env vars | Internal ops channel ID |
| Telegram credential | n8n credentials store | Bot token attached to the `Alert Ops Channel` node |

## Setting up the Shopify webhook

After importing the workflow:

1. Open the **Shopify orders/create Webhook** node and copy the **Test URL** (or **Production URL** when ready). It looks like:
   `https://your-n8n.example.com/webhook/shopify-order-created`
2. In Shopify Admin → **Settings → Notifications → Webhooks**, create a new webhook:
   - Event: `Order creation`
   - Format: `JSON`
   - URL: paste from step 1
3. Save. Shopify will send a test payload — confirm it arrives in n8n's webhook log.

## Edge cases handled

- **The `total_price` field arrives as a string** from Shopify. The Set node parses it to a number via `parseFloat(...)` so the IF node's numeric comparison works correctly.
- **Missing `source_name`.** Default to `'web'` so Telegram alerts never blow up with `undefined`.
- **Both Klaviyo branches converge on `Respond 200`.** This guarantees Shopify always gets an ACK even if the Klaviyo POST fails — the webhook is considered delivered so Shopify doesn't retry and create duplicate side-effects.

## Limitations (and how to extend)

- **No retry on Klaviyo failure.** If the Klaviyo POST 5xx's, this workflow logs the error but doesn't retry. For production, wrap the Klaviyo HTTP node in n8n's built-in retry (`options → retry on fail`) or fan out to a dead-letter queue.
- **Klaviyo profile-import is "upsert by email" only.** If two storefronts share an email but want different segments, you'll need a different Klaviyo identifier (phone number or external ID).
- **The 200 EUR threshold is hardcoded.** For multi-currency stores, convert via `currency` first (e.g. exchange-rate HTTP call before the IF node).

## How to import

1. In n8n, **Workflows → Import from File** → select [`workflows/shopify-order-pipeline.json`](../workflows/shopify-order-pipeline.json).
2. Replace Telegram credential placeholder with your bot credential.
3. Set the env vars above.
4. Register the webhook in Shopify (see "Setting up the Shopify webhook" above).
5. Activate the workflow.
