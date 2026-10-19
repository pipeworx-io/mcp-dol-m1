# @pipeworx/dol-m1 — Form M-1 (MEWA/AHP) filings

U.S. Department of Labor Form M-1 filings — the federal report filed by
multiple employer welfare arrangements (MEWAs) and association health plans
(AHPs) that pool health coverage across unrelated employers. `form5500`
already covers annual plan/pension financial returns, and `ebsa` covers
EBSA's own civil-penalty enforcement record against deficient Form 5500
filers — this is a third, unrelated filer population and form.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1843+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

| Tool | Answers |
|---|---|
| `dol_m1_search` | Search filings by plan year, MEWA/sponsor name, EIN, headquarters state, state(s) where coverage is offered, filing type, stop-loss, fully-insured |
| `dol_m1_filer` | Full filing history (every plan year filed) for one named MEWA/sponsor or EIN |

## Auth

Keyless. No login, no API key, no approval step.

## Data sources

- U.S. Department of Labor, Employee Benefits Security Administration — Form
  M-1 Filings Database. 17,313 filings live as of 2026-10-18, plan years
  2003-2025.

## Gotchas

- **Reverse-engineered, not documented.** This is a public search tool, not a
  published REST API — the endpoint shape came from the search page's own
  bundled JavaScript (`m1search.build_url`), not from any API docs. It is
  still a plain, publicly reachable `GET` with no auth; nothing here needs a
  browser or a session.
- **The dropdown filters do not accept an empty string the way a free-text
  field does.** Plan year and filing type default to `"ALL"`, and the
  stop-loss/fully-insured flags default to `"0"` ("Any") in the real form.
  Sending `""` for any of those four throws an unhandled server exception —
  the endpoint answers **HTTP 500 with a full HTML error page**, not a
  structured error. This pack always sends the correct default.
- **Headquarters state and coverage state are different facets, not
  aliases.** `hq_state` filters where the MEWA is based; `coverage_state`
  filters where it offers coverage — a MEWA headquartered in New Jersey can
  show up filtered by `coverage_state: "NY"` with `hq_state: "NJ"` on its own
  row. Verified live.
- **`limit` has an undocumented cap of 100, and it does not clamp or error
  past that — it silently resets to a 25-row default.** Measured live:
  `limit=100` returns 100 rows, `limit=101` returns 25. This pack clamps its
  own `limit` argument to 1-100 before calling, so a caller's `limit: 500`
  cannot trigger the silent fallback.
- **A zero-match query is a clean `HTTP 200` with `{"total":0,"rows":[]}`** —
  not an error. This pack turns that into `{ found: false, reason:
  "no_matching_filings", filters_applied: {...} }` rather than passing
  through a bare `[]`.
- **The `FilingType` field on each returned row does not reliably reflect
  the `filing_type` request filter** — every row observed across every
  filing-type query (Amended, Extension, Regular Filing, Registration) came
  back `FilingType: 0`. The request-side filter itself works (verified live:
  filtering narrows the total count correctly per type); only the echoed
  field is unreliable, so it is left out of the shaped output.
- **The administrator/sponsor name can be an individual**, not a business.
  For small MEWAs the Form M-1 administrator of record is sometimes a
  natural person's own name — this is the DOL's own mandatory public-filing
  field, passed through exactly as the federal filing discloses it, the same
  way `form5500` and `ebsa` already pass through their own sponsor fields.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "dol-m1": {
      "url": "https://gateway.pipeworx.io/dol-m1/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/dol-m1/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1843+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/dol_m1_search \
  -H 'Content-Type: application/json' \
  -d '{"hq_state":"TX","plan_year":2024,"limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/dol_m1_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "dol-m1": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-dol-m1"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-dol-m1
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Dol M1 data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
