# n8n-workflows-catalog

> Production-ready n8n workflows for e-commerce ops. Three workflows covering scheduled briefings, webhook-driven order pipelines, and LLM-based content localization.

This is a personal portfolio project — generic automation patterns I've found useful, packaged so anyone can import and adapt them. The workflows aren't toy demos: they handle edge cases (rate limits, retries, missing fields, MarkdownV2 escaping), use real APIs (Anthropic, Telegram, Klaviyo, Shopify webhooks), and are credential-placeholder-only so they import cleanly into any n8n instance. No client data or proprietary content is included.

---

## Workflows

### 1. [Daily Trading Brief](./docs/daily-trading-brief.md)

**Schedule → Reddit + RSS → Claude → Telegram**

Pulls the top of `r/wallstreetbets` plus Investing.com headlines every weekday at 07:30, sends the combined corpus to Claude Haiku 4.5 for a 4-6 bullet pre-market summary, and posts it to a Telegram channel.

| Trigger | Services | Cost (monthly) |
|---|---|---|
| `scheduleTrigger` cron `30 7 * * 1-5` | Reddit JSON · Investing.com RSS · Anthropic Messages API · Telegram Bot | < $0.05 at Haiku rates |

→ JSON: [`workflows/daily-trading-brief.json`](./workflows/daily-trading-brief.json) · Doc: [`docs/daily-trading-brief.md`](./docs/daily-trading-brief.md)

### 2. [Shopify Order Pipeline](./docs/shopify-order-pipeline.md)

**Webhook → Shopify order → Klaviyo + ops alert**

Receives `orders/create` webhooks from Shopify, extracts key fields, branches on order total. High-value orders fire a Telegram alert to the ops channel and tag the customer in Klaviyo as `vip-prospect`. Standard orders tag as `new-customer`. Always responds 200 OK so Shopify doesn't retry.

| Trigger | Services | Notes |
|---|---|---|
| `webhook` POST endpoint | Shopify Admin · Klaviyo Profile API · Telegram Bot | Synchronous, sub-second response time |

→ JSON: [`workflows/shopify-order-pipeline.json`](./workflows/shopify-order-pipeline.json) · Doc: [`docs/shopify-order-pipeline.md`](./docs/shopify-order-pipeline.md)

### 3. [AI Content Localization](./docs/ai-content-localization.md)

**Manual → Shopify CSV → Claude translate (structured outputs) → Shopify CSV**

Reads a Shopify product export CSV, translates Title / Body (HTML) / SEO Title / SEO Description into a target language via Claude Sonnet 4.6, and writes an import-ready CSV with all other columns preserved. Uses Claude's tool_use feature for guaranteed-structured output — no regex-parsing of freeform translations.

| Trigger | Services | Capacity |
|---|---|---|
| `manualTrigger` | File system (CSV in/out) · Anthropic Messages API with tool_use | ~$3-5 per 1000-product catalog per language pair |

→ JSON: [`workflows/ai-content-localization.json`](./workflows/ai-content-localization.json) · Doc: [`docs/ai-content-localization.md`](./docs/ai-content-localization.md)

---

## Quick start

### Prerequisites

- An n8n instance (self-hosted or [n8n Cloud](https://n8n.io/cloud/)).
- Credentials for the services each workflow uses — see individual workflow docs for which apply.

### Import a workflow

1. In your n8n UI: **Workflows → Import from File**.
2. Choose any `.json` from [`workflows/`](./workflows/).
3. Open each node that uses external credentials (Telegram, Klaviyo, etc.) and replace the placeholder credential ID with one you've configured in **Credentials**.
4. Set the env vars listed in `.env.example` (n8n: **Settings → Environments**, or via `docker-compose` env block / Kubernetes secret).
5. Toggle the workflow active.

### Test locally with `docker-compose`

Spin up n8n + the env vars from `.env.example`:

```bash
docker run -d --name n8n -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  --env-file .env \
  n8nio/n8n:latest
```

Open http://localhost:5678 → import any workflow from this repo.

---

## Architecture notes

Patterns repeated across all three workflows:

- **Credentials in n8n's credential store, secrets via env vars.** API keys (Anthropic, Klaviyo) are referenced as `{{ $env.NAME }}` in HTTP Request node headers — never inlined in JSON. Telegram and OAuth-style credentials use n8n's native credential UI.
- **Structured outputs via Claude `tool_use` for LLM responses that downstream logic depends on.** Free-form LLM output breaks in dozens of subtle ways (code-fence wrapping, preambles, swapped fields). The localization workflow demonstrates a `submit_translation` tool with a strict JSON schema — Claude is forced to call it via `tool_choice`, and the response is always validated JSON.
- **Webhook responses converge through `respondToWebhook`** so external systems (Shopify) always get 200 OK regardless of which branch ran. The webhook is considered delivered — no retries — even if a downstream POST fails. Errors get logged and (optionally) routed to an error-handler workflow.
- **`splitInBatches` for rate-limit-safe iteration** over large datasets. The localization workflow processes 5 rows at a time, well below Claude tier-1 rate limits (~50 req/min), and scales to ~10k-product catalogs at acceptable wall-clock time.
- **Defensive field extraction** at the boundaries: every payload is treated as untrusted. The order pipeline parses `total_price` as a float because Shopify ships it as a string. The Reddit cleanup code defaults `children || []` because the API can return empty.

---

## What's not here (yet)

- **MCP servers / Anthropic agentic patterns.** Roadmap — wiring an MCP-based sub-process into the order pipeline so the LLM can call out to validators (price-range sanity, inventory checks) before the workflow responds to Shopify.
- **n8n + LangChain agent workflows.** Useful for multi-step reasoning, but currently the use cases in this catalog are single-call + structured output. Not yet a fit.
- **CI for workflow JSON.** A small GitHub Action that runs JSON validation + a smoke test would be the natural next step.

---

## License

MIT — see [LICENSE](./LICENSE). Use these workflows, fork them, adapt them to your store.
