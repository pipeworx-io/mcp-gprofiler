# @pipeworx/gprofiler

Functional enrichment for a gene list against GO, KEGG, Reactome, WikiPathways, TRANSFAC,
miRTarBase, CORUM, HPA and HPO, plus gene/protein identifier conversion and cross-species ortholog
mapping — from g:Profiler at the University of Tartu.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `gprofiler_enrich(organism, query[], sources?, user_threshold?, no_iea?, ordered?, domain_scope?, background?, significant?, limit?)`
  — g:GOSt. Returns each enriched term with an ALREADY-ADJUSTED p-value (g:SCS), term size, query
  size, overlap, precision and recall. Per-gene `intersections` and ontology `parents` are dropped:
  they dominate the payload and nothing reads them in an answer.
- `gprofiler_convert_ids(organism, query[], target?, numeric_namespace?)` — g:Convert. Maps between
  gene/protein/transcript/probe namespaces and reports `hitsForInput`, so an ambiguous symbol shows
  up as ambiguous instead of quietly resolving to one gene.
- `gprofiler_orthologs(organism, target, query[])` — g:Orth via Ensembl Compara. Human TP53 →
  mouse Trp53, with one-to-many mappings flagged rather than collapsed.

## Auth

Keyless. g:Profiler asks programmatic callers to identify themselves; the pack sends a
`pipeworx-mcp-gprofiler` User-Agent.

## Data sources

- `POST https://biit.cs.ut.ee/gprofiler/api/gost/profile/` — enrichment.
- `POST https://biit.cs.ut.ee/gprofiler/api/convert/convert/` — ID conversion.
- `POST https://biit.cs.ut.ee/gprofiler/api/orth/orth/` — orthologs.
- Docs: <https://biit.cs.ut.ee/gprofiler/page/apis>

## Traps

**The organism code is g:Profiler's own and nothing else works.** First letter of the genus plus the
full species name, lowercase: `hsapiens`, `mmusculus`, `rnorvegicus`, `drerio`, `dmelanogaster`,
`celegans`, `scerevisiae`, `athaliana`. `"human"` and `"9606"` are both rejected. This is the
common first failure, and it is a loud one, which is the good case.

**A single gene passed as a bare string would be split into characters upstream.** The pack accepts
a string and splits it on whitespace/comma/semicolon before sending, so `"TP53"` becomes
`["TP53"]` rather than four failed lookups returned as a clean 200.

**An empty enrichment result is ambiguous and the pack says so.** No significant term is a real
answer for a small or functionally unrelated list — and it is also exactly what a wrong organism
code or unrecognised identifiers produce. The `note` field points the caller at
`gprofiler_convert_ids` to tell the two apart.

**`pValue` is already multiple-testing corrected** (g:SCS by default). Do not correct it again.

**`n_incoming > 1` on a conversion means the INPUT matched more than one record**, not that the
output is multi-valued. Surfaced as `hitsForInput` and collected in `ambiguousInputs`.

**Enrichment against all sources on a large list is genuinely slow** (seconds, not milliseconds).
Pass `sources` when you know which annotation set you want.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "gprofiler": {
      "url": "https://gateway.pipeworx.io/gprofiler/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/gprofiler/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/gprofiler_enrich \
  -H 'Content-Type: application/json' \
  -d '{"organism":"hsapiens","query":["TP53","BRCA1","BRCA2","ATM","CHEK2","PALB2"],"sources":["REAC"],"limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/gprofiler_enrich`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-gprofiler` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/gprofiler/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "gprofiler": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-gprofiler"]
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
ask_pipeworx({ question: "your question about Gprofiler data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
