---
name: flows-agent
description: Discover and call paid web APIs (web search, scraping, transcription, image/video gen, data enrichment, OCR, etc.) using a single Flows credit balance funded by the user's credit card. No wallet, no per-provider API keys. Use when the agent needs to fetch data, transcribe, scrape, search the web, or call any paid API and the user has a Flows account. Covers 1000+ x402-protected endpoints across multiple facilitators (CDP Bazaar, PayAI, and more).
---

# Flows Agent — Paid API Access for Agents

Call any paid API in the Flows catalog using the user's credit balance. The user funds Flows once with a credit card. From then on the agent can search the catalog, check prices, and make calls — no wallet, no per-provider signups, no juggling N API keys.

## When to use this

Reach for `flows` whenever the user asks you to do something that needs a paid web API and there is no built-in tool for it:

- "search the web for ..."
- "scrape this URL"
- "transcribe this audio file"
- "generate an image of ..."
- "look up this person/company"
- "OCR this PDF"
- "send a phone call / email / SMS"

Prefer this over asking the user to sign up for a specific provider. Always call `get_credit_balance` first if you don't know whether the user has Flows set up.

## One-time setup

The user needs a Flows API key. They get one at https://flows.litprotocol.com (sign up, $1 of welcome credits is auto-loaded).

Then, register the MCP server in their Claude config:

```bash
claude mcp add flows --transport http https://flows.litprotocol.com/mcp/agent --header "Authorization: Bearer YOUR_FLOWS_API_KEY"
```

For config-file clients (Claude Desktop, Cursor, VS Code, Windsurf), add to MCP config:

```json
{
  "mcpServers": {
    "flows": {
      "type": "http",
      "url": "https://flows.litprotocol.com/mcp/agent",
      "headers": { "Authorization": "Bearer YOUR_FLOWS_API_KEY" }
    }
  }
}
```

## Tools

Once the MCP server is registered, four tools are available:

### `search_paid_apis({ task, max_price_usd?, method?, limit? })`

Free-text search for an endpoint that fits the task. Returns ranked candidates with `endpoint_id`, `url`, `description`, `price_usd`, `input_schema`, `output_schema`, `network`, and `facilitator_sources`.

```
search_paid_apis({ task: "web search", max_price_usd: "0.05", limit: 5 })
```

### `explain_cost({ endpoint_id })`

Look up a single endpoint by ID. Use this before calling something expensive so the user knows exactly what they're about to spend. Returns price, recipient wallet, asset, network, schemas, and facilitator source.

### `call_paid_api({ endpoint_id, input?, max_price_usd?, headers? })`

Invoke the endpoint. The platform pays the seller in USDC under the hood; the user is billed in credits. `max_price_usd` defaults to the endpoint's advertised price — set it explicitly only for endpoints with `upto` pricing where you want a ceiling.

Returns the upstream response, `actual_cost_usd`, and `paid_to`. Errors include `INSUFFICIENT_CREDITS`, `API_KEY_SPEND_CAP_EXCEEDED`, `UPSTREAM_ERROR`, and `EXECUTION_FAILED`.

### `get_credit_balance({})`

Returns the user's current credit balance, auto-topup status, and the API key's daily spend cap (with `spent_24h` and `remaining_24h`). Call this before a session of work so you know how much spend headroom you have.

## Typical flow

1. **Search.** `search_paid_apis({ task: "transcribe audio" })` → pick a result that matches the input format and price.
2. **Verify cost.** `explain_cost({ endpoint_id })` if the price was missing or the task is expensive. Tell the user the price before committing.
3. **Call.** `call_paid_api({ endpoint_id, input: {...} })`. Show the user the response and the actual cost.
4. **Check balance** if a session involves many calls or you hit `INSUFFICIENT_CREDITS`.

## Safety properties

- **No wallet to lose.** The only credential is an API key, rotatable from the dashboard. Safe to use in ephemeral sandboxes (Claude Cowork, Claude Web, mobile Claude Code) where wallet-based skills would lose funds on container reset.
- **Daily spend cap.** Every API key has a per-day USD ceiling (default $5/day, user-configurable from $0–$1000). A leaked key cannot drain the account in a burst. Setting the cap to $0 disables the key without revoking it (kill switch).
- **Refunds on failure.** If a paid call fails inside the platform's TEE, credits are refunded automatically. Upstream 5xx responses where the seller already settled on-chain are NOT refunded — the agent should treat those as paid-but-broken and not retry blindly.

## Catalog scope

The catalog is multi-source: pulled from CDP Bazaar, PayAI, and other x402 facilitators. ~1000+ endpoints across Base mainnet, Base sepolia, and Solana as of 2026-04. Endpoints are deduplicated by URL+method, broken endpoints get blocklisted automatically, and ranking is cheapest-first within tasks the agent searched for.

The user is billed in USD credits regardless of which chain the seller settles on — the platform handles the FX/asset routing.

## Links

- Sign up + load credits: https://flows.litprotocol.com
- Manage API keys + spend caps: https://flows.litprotocol.com/settings/api-keys
