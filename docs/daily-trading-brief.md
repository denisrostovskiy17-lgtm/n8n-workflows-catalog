# Daily Trading Brief

A scheduled workflow that produces a concise pre-market trading brief every weekday at 07:30 Warsaw time and posts it to a Telegram channel.

## What it does

1. Triggers on a cron schedule (07:30 Mon–Fri).
2. Pulls today's top posts from `r/wallstreetbets` and the latest financial headlines from Investing.com's RSS feed in parallel.
3. Merges and cleans the two streams (strips HTML, dedupes, caps length to keep prompt costs predictable).
4. Sends the combined corpus to Claude Haiku 4.5 with a system prompt instructing it to write a 4-6 bullet pre-market brief: overnight mood, named tickers with chatter, macro events to watch, one contrarian read.
5. Formats the result as Telegram Markdown and posts to a configured channel.

## Nodes

| # | Node | Type | Purpose |
|---|---|---|---|
| 1 | Schedule (07:30 Mon–Fri) | `scheduleTrigger` | Cron `30 7 * * 1-5` |
| 2 | Fetch r/wallstreetbets | `httpRequest` | Reddit JSON API, top of day, limit 15 |
| 3 | Fetch Investing.com RSS | `httpRequest` | News RSS, raw XML |
| 4 | Merge Sources | `merge` | Combine the two HTTP outputs |
| 5 | Clean & Combine | `code` | Strip HTML, build the prompt corpus |
| 6 | Claude — Summarize | `httpRequest` | POST `/v1/messages` (Claude Haiku 4.5) |
| 7 | Format for Telegram | `set` | Markdown wrapper around Claude's text |
| 8 | Send to Telegram | `telegram` | Post to channel |

## Required credentials and env vars

| Variable | Where | Notes |
|---|---|---|
| `ANTHROPIC_API_KEY` | n8n env vars | Used inline in the Claude HTTP node header |
| `TELEGRAM_CHAT_ID` | n8n env vars | Channel/chat ID. Negative for channels (e.g. `-100…`) |
| Telegram credential | n8n credentials store | Bot token, attached to the `Send to Telegram` node |

## Edge cases handled

- **Reddit rate limits.** Reddit's public JSON endpoint allows ~60 req/min unauthenticated; one daily run is well within limits. A custom `User-Agent` header is set to avoid being throttled as anonymous.
- **Empty Reddit / RSS responses.** The cleaning code defensively pulls `children || []` and parses regex matches that may return zero items — Claude still receives a coherent prompt with one empty section.
- **Claude API errors.** The HTTP Request node surfaces 4xx/5xx as a workflow error, which n8n logs and (optionally) emits to a configured error workflow. Add an Error Trigger workflow to ping ops on failure.
- **Telegram MarkdownV2 escaping.** This workflow uses `parse_mode: Markdown` (the legacy mode) for simplicity. If you switch to MarkdownV2, special characters `_*[]()~`>#+-=|{}.!` must be escaped.

## Cost estimate

- Claude Haiku 4.5: ~$0.25/$1.25 per 1M input/output tokens.
- One run sends ~2-3K input tokens and gets ~400 output tokens.
- Monthly cost (20 weekdays): roughly $0.02. Effectively free.

## How to import

1. In n8n, open **Workflows → Import from File**.
2. Select [`workflows/daily-trading-brief.json`](../workflows/daily-trading-brief.json).
3. Open each Claude/Telegram node and replace credential placeholders with your own.
4. Set the env vars listed above (Settings → Environments, or via `docker-compose` env block).
5. Toggle the workflow active.

## Customization ideas

- Swap the Reddit subreddit (e.g. `r/stocks`, `r/options`).
- Add a second LLM step for a "today's risks" sub-section.
- Replace Telegram with Slack, Discord, or email (`emailSend` node).
- Switch to `claude-sonnet-4-6` for higher-quality summarization at ~10x the cost (still under $1/month at this volume).
