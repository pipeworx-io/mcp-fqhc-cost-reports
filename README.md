# @pipeworx/fqhc-cost-reports

Per-center, per-fiscal-year Medicare cost report figures for federally qualified health centers (FQHCs): visits, total allowable cost, cost per visit, Medicare visits and cost, and visits by payer (Medicare, Medicaid, other), from CMS cost report form CMS-224-14, fiscal years 2015 onward.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 2006+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `fqhc_cost_report_search(name?, state?, city?, fiscal_year?, sort?, limit?)` -- find centers and compare cost per visit; sort by name, cost per visit or visits. Unmatched filters return near-name centers or the valid states and years.
- `fqhc_cost_report_center(provider_number | name + state)` -- one center across every fiscal year, with the year-over-year change in cost per visit and Medicare/Medicaid visit shares.
- `fqhc_cost_report_state_summary(state, fiscal_year?)` -- centers reporting, total visits, total allowable cost, aggregate and median cost per visit, lowest and highest cost centers, Medicare and Medicaid shares.

## Auth

Keyless.

## Data sources

- Centers for Medicare & Medicaid Services (CMS), Healthcare Cost Report Information System, Federally Qualified Health Center cost reports (form CMS-224-14). US federal data, public domain. A stored, derived copy: one row per center per fiscal year (about 20,000 rows, 3.5 MB), refreshed by re-running `scripts/bake-index.mjs` on CMS's current release. Every answer carries `data_as_of` (the latest CMS processing date in the release).

Where each figure comes from on the form (also recorded in `scripts/bake-index.mjs`):

| Field | Worksheet, line, column |
|---|---|
| total_allowable_cost | Worksheet B Part I, line 11, column 5 (total costs by practitioner) |
| cost_per_visit | Worksheet B Part I, line 13, column 6 |
| medical / mental health / IOP visits | Worksheet B Part I, line 11, columns 7 / 8 / 8.01 |
| medicare_visits, medicare_cost | Worksheet B Part I, line 11, columns 9+10+10.01 and 11+12+12.01 (Title XVIII) |
| visits by payer | Worksheet S-3 Part I, lines 20 + 21 + 25, columns 1 Title V, 2 Title XVIII (Medicare), 3 Title XIX (Medicaid), 4 other, 5 all patients |
| total_expenses_all_cost_centers | Worksheet A, line 100, column 7 |
| name, city, state, zip | Worksheet S-1 Part I, lines 1 and 3 |

Things worth knowing:

- Fiscal year means the year the cost reporting period ends. Periods can be shorter than 12 months (a center's first or last period), so compare visits and cost, not just totals, across years.
- Cost per visit is the form's average cost per visit, not a payment rate. A few centers report implausible values (for example a center filing only mental health visits against its whole cost); the figures are as filed.
- A report filed for several sites together appears once, under the filing provider number. Sites covered by another entity's consolidated report carry no figures and are not listed (about 11,700 such records were left out of the 31,700 in the release).
- If a center filed more than once for the same period, the most recently processed report is kept.
- Provider numbers (CCN) are 6-character strings; keep leading zeros. HRSA site data (`hrsa-health-centers`) uses different identifiers, so there is no crosswalk here.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "fqhc-cost-reports": {
      "url": "https://gateway.pipeworx.io/fqhc-cost-reports/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/fqhc-cost-reports/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 2006+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/fqhc_cost_report_search \
  -H 'Content-Type: application/json' \
  -d '{"state":"OH","fiscal_year":"2022","sort":"cost_per_visit_desc","limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/fqhc_cost_report_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-fqhc-cost-reports` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/fqhc-cost-reports/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "fqhc-cost-reports": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-fqhc-cost-reports"]
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
ask_pipeworx({ question: "your question about Fqhc Cost Reports data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
