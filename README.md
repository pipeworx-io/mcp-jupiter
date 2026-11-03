# @pipeworx/jupiter

Cross-DEX Solana token price and swap-route quotes from Jupiter — the
aggregator that reads liquidity across every DEX/AMM on Solana and returns a
single best-price view and a best-route swap quote, rather than one pool's
view.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1867+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `jupiter_price(mints)` — USD price, 24h price change, and aggregated
  liquidity for one or more Solana token mints (up to 50 per call).
- `jupiter_quote(input_mint, output_mint, amount, slippage_bps?)` — the best
  swap route between two Solana token mints: expected output amount, price
  impact, and which AMMs/DEXes the route crosses.

## Auth

Keyless. No activation gate.

## Data sources

- Jupiter, Jupiter Aggregator price and quote service — aggregates price and
  liquidity across Solana DEXes/AMMs and computes best-route swap quotes.
  Routes live only in `src/index.ts`.

## Notes

- `jupiter_price` returns `found: false` for a mint entry rather than erroring
  when Jupiter has no price data for that mint (e.g. too new or too
  illiquid to have been indexed yet) — a 200 with a per-mint miss flag, not a
  blanket failure for the whole batch.
- `jupiter_quote`'s `amount` is in the INPUT token's base units (e.g.
  lamports for SOL — 1 SOL = 1,000,000,000), not a human-readable token
  count. Decimals vary per token; callers must know the input token's
  decimals (or query `jupiter_price` first, which returns `decimals`).
- This is a cross-DEX AGGREGATE price/route, distinct from the `dexscreener`
  pack's per-pool view — use `dexscreener` to inspect one specific pool
  (e.g. a brand-new pump.fun pool before Jupiter has indexed it), and this
  pack for "what would I actually get swapping," which is what Jupiter's
  router is built to answer.
- Out of scope by decision (fleet #3986): anything "sniping"-shaped — there
  is no sanctioned API for a pump.fun launch feed, and transaction-landing /
  bundle services (Jito and similar) are paid execution infrastructure, not
  data; this gateway is request/response, not a trading execution path.
- Jupiter's free tier rate-limits by IP on a short rolling window; this pack
  makes exactly one outbound call per tool call (no fan-out).

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "jupiter": {
      "url": "https://gateway.pipeworx.io/jupiter/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/jupiter/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1867+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/jupiter_price \
  -H 'Content-Type: application/json' \
  -d '{"mints":["So11111111111111111111111111111111111111112","EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v","Ez1zGM23iZgzxvinkNEnTuSkyicwbahvRjfpRDaWnzbj"]}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/jupiter_price`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-jupiter` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/jupiter/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "jupiter": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-jupiter"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Jupiter data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
