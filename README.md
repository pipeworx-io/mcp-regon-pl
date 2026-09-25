# @pipeworx/regon-pl

Polish business register (REGON / BIR1.1) — resolve any Polish company by NIP,
REGON or KRS number to its registered name, legal address and entity type, from
Statistics Poland's official register.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1679+ live data sources.

## Tools

- `regon_search_nip({ nip, environment, _apiKey })` — by tax identification number.
- `regon_search_regon({ regon, environment, _apiKey })` — by 9- or 14-digit REGON statistical number.
- `regon_search_krs({ krs, environment, _apiKey })` — by National Court Register number.

All three return the registered name, the other two identifiers, entity type
(legal person / sole trader / local unit), voivodeship, county, municipality,
locality, postcode and registered address.

## Auth

Platform key (`PLATFORM_REGON_KEY`) with BYO override via `?_apiKey=`, **plus a
credential-free test path**.

- `environment: "production"` (default) — the live register. Requires an API
  key. Register free at <https://api.stat.gov.pl/Home/RegonApi>; GUS issues a
  20-character user key.
- `environment: "test"` — GUS's own public test service with the test key GUS
  publishes in its documentation. Needs no credential and genuinely answers,
  but from a small fixed fixture database (addresses read "ul. Test-Wilcza").
  Use it to check the response shape, never for a fact about a real company.
  Every response from it carries a `disclaimer` field saying so.

## Data sources

- <https://wyszukiwarkaregon.stat.gov.pl/wsBIR/UslugaBIRzewnPubl.svc> — production BIR1.1.
- <https://wyszukiwarkaregontest.stat.gov.pl/wsBIR/UslugaBIRzewnPubl.svc> — test BIR1.1.

## Traps

- **SOAP 1.2 over MTOM, not REST.** Two calls per query: `Zaloguj` exchanges
  the user key for a session id, then the search carries it.
- **The session id goes in an HTTP `sid` header, not in the SOAP envelope.**
  Put it in the envelope and the service authenticates you as nobody and
  returns an empty result *with no error*.
- **A no-match is not an empty document.** It is a populated `<dane>` record
  whose only fields are `ErrorCode` 4 and `ErrorMessageEn` "No data found for
  the specified search criteria". Parsed naively that is one result with every
  field null — a textbook silent zero. This pack detects `ErrorCode` and
  reports `found: false` with the upstream message.
- **An unrecognised user key is reported as an empty `ZalogujResult`**, not as
  an HTTP error. The pack raises on it explicitly.
- Responses are MIME-multipart with the payload as an XML-escaped string inside
  the SOAP result element — it needs unwrapping twice.
- Sessions are cached in worker memory for 30 minutes (upstream allows ~60).

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "regon-pl": {
      "url": "https://gateway.pipeworx.io/regon-pl/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/regon-pl/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1679+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/regon_search_nip \
  -H 'Content-Type: application/json' \
  -d '{"nip":"7740001454","environment":"test"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/regon_search_nip`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "regon-pl": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-regon-pl"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-regon-pl
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Regon Pl data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
