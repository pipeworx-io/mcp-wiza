# @pipeworx/wiza

Find a person's mobile phone number from their LinkedIn profile URL, via
Wiza's individual-reveal API (wiza.co). Part of `enrich-waterfall`'s new
`enrich_find_phone` finder rotation (fleet #2860).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1765+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `wiza_find_phone(profile_url, _apiKey)` — starts a reveal and polls it
  inline (bounded ~10s across 5 polls) so the caller gets one synchronous
  answer instead of a bare reveal id to manage.

## Auth

BYO only. Pass your Wiza API key as `_apiKey` (header
`Authorization: Bearer <key>`). Find your key on your Wiza account API
page. No platform key (D4).

## Data sources

- `https://wiza.co/api/individual_reveals` — start a reveal (POST) and poll
  it (GET `/individual_reveals/{id}`), with `enrichment_level: "phone"`.

## Verification notes (fleet #2860, 2026-10-08, no vendor key held)

- **Edge-probed live with an invalid key**: HTTP 401
  `{"status":{"code":401,"message":"Invalid API key."}}` — confirms the
  endpoint, the `Authorization: Bearer` header, and that a bad key fails
  cleanly.
- **The reveal is genuinely async** (Wiza's own docs describe "0-15
  seconds" typical, "~4 seconds" average) — this pack polls up to 5 times
  with a 2s gap inside one tool call. We hold no Wiza key, so **this poll
  loop and its field parsing (`mobile_phone`/`phone_number`/`phones[]`,
  `phone_status`) were never exercised against a real reveal** — they are
  built directly from https://docs.wiza.co/using-the-api/find-mobile-phone-number
  (read 2026-10-08). If a real reveal returns a different field name for the
  number, the loop will exhaust its 5 attempts and report a `pending: true`
  not-found rather than a confirmed miss — that distinction is deliberate
  (see below) but depends on `phone_status` actually appearing as
  documented.
- **Pending vs. confirmed-miss is a real distinction, not padding.** If the
  5-poll budget runs out without Wiza setting `phone_status: "not_found"`,
  this pack returns `{ found: false, pending: true, reveal_id }` rather than
  a flat miss, because Wiza may simply still be working — a caller (or the
  waterfall) can decide whether to wait longer rather than being told
  falsely that no phone exists.
- **Not verified (no key held):** the exact `phone_status` enum; whether
  `phones[]` can return more than one number worth preferring by type.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "wiza": {
      "url": "https://gateway.pipeworx.io/wiza/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/wiza/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1765+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

This pack takes your own API key (`_apiKey`) — we don't front one for it, so there's no curl here that would run without it. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/wiza_find_phone`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "wiza": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-wiza"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-wiza
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Wiza data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
