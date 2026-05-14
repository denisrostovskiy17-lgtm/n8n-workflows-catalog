# AI Content Localization (Shopify CSV)

Translates Shopify product catalogs across languages via Claude API, preserving HTML structure and SEO fields, in batches of 5 so it stays within rate limits on large catalogs.

## What it does

1. Triggers manually (you press **Execute Workflow**).
2. Reads a Shopify product export CSV from a configurable path.
3. Parses the CSV into rows.
4. Processes rows in batches of 5:
   - For each row, sends `Title`, `Body (HTML)`, `SEO Title`, `SEO Description` to Claude Sonnet 4.6.
   - Forces Claude to respond via a `submit_translation` tool whose JSON schema is the four translated fields — no regex-parsing of freeform output.
   - Merges the translated fields back into the original row, keeping `Handle`, `Vendor`, `Type`, `Tags`, `Variant Price`, etc., untouched.
5. After the loop completes, writes all rows to a target CSV in import-ready Shopify format.

## Nodes

| # | Node | Type | Purpose |
|---|---|---|---|
| 1 | Manual Trigger | `manualTrigger` | Run on demand |
| 2 | Config | `set` | Input/output paths and target language |
| 3 | Read CSV File | `readWriteFile` | Load binary from disk |
| 4 | Parse CSV → Rows | `spreadsheetFile` | Binary → JSON rows |
| 5 | Loop in Batches of 5 | `splitInBatches` | Rate-limit-safe iteration |
| 6 | Claude — Translate | `httpRequest` | POST `/v1/messages` with structured output |
| 7 | Merge Translation | `code` | Extract tool_use result, merge into row |
| 8 | Rows → CSV | `spreadsheetFile` | JSON → CSV binary |
| 9 | Write Output CSV | `readWriteFile` | Save binary to disk |

## Required credentials and env vars

| Variable | Where | Notes |
|---|---|---|
| `ANTHROPIC_API_KEY` | n8n env vars | Used inline in the Claude HTTP node header |
| Input CSV file | Mounted volume / file path | Default: `/data/shopify-products-en.csv` (set in `Config` node) |

## Why structured outputs (tool_use) here

Free-form translation responses break in dozens of subtle ways: Claude wrapping output in code fences, adding "Here is your translation:" preambles, swapping the order of fields, hallucinating quotation marks inside HTML attributes.

This workflow uses Claude's **tool_use** with a Pydantic-style JSON schema:

```json
{
  "name": "submit_translation",
  "input_schema": {
    "type": "object",
    "properties": {
      "title":           { "type": "string" },
      "body_html":       { "type": "string" },
      "seo_title":       { "type": "string" },
      "seo_description": { "type": "string" }
    },
    "required": ["title", "body_html", "seo_title", "seo_description"]
  }
}
```

With `tool_choice: { type: "tool", name: "submit_translation" }`, Claude is forced to call exactly this tool. The response is always validated JSON — the `Merge Translation` node trusts the structure and merges directly into the row.

## Edge cases handled

- **HTML inside `Body (HTML)`** is preserved by instructing Claude in the system prompt and trusting structured output. If a translation breaks the HTML, the import to Shopify would fail loudly rather than silently produce mangled product pages.
- **Missing SEO fields.** The HTTP body falls back to `""` if `SEO Title` or `SEO Description` are absent in the source row — Claude returns empty strings for empty inputs.
- **Batch size of 5** is conservative for Claude tier-1 rate limits (50 requests/minute). For tier-2+ accounts, bump to 25-50 for ~5x faster runs on a 1000-product catalog.
- **Large catalogs.** A 1000-product catalog at batch=5 takes ~6-8 minutes (one batch of 5 takes ~3-4s of total Claude latency). For 10k+ catalogs, switch to Anthropic's [Message Batches API](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing) and submit in async batches of 1000.

## Cost estimate

- Claude Sonnet 4.6: $3 / $15 per 1M input/output tokens.
- Average product row: ~400 input tokens (title + body), ~400 output tokens (translation back).
- A 1000-product catalog: ~$3-5 per language pair. Single full localization (EN→DE) typically under $5.

## How to import

1. **Workflows → Import from File** → select [`workflows/ai-content-localization.json`](../workflows/ai-content-localization.json).
2. Open the **Config** node and set `input_path`, `output_path`, and `target_lang` for your run.
3. Set `ANTHROPIC_API_KEY` env var.
4. Mount the input CSV into the n8n container at the path you set in step 2.
5. Run via **Execute Workflow** (the trigger is manual).

## Customization ideas

- Run multiple target languages in a single workflow by adding a fan-out before `Config` (one branch per language).
- Add a glossary system prompt that forces Claude to translate brand-specific terms a certain way ("ATB Holdings" → never translate; "Premium Selection" → "Auswahl Premium" in German).
- Cache translations by hash of source text + target language to avoid re-paying for unchanged products on subsequent runs.
