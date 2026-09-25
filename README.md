# @pipeworx/gprofiler

Functional enrichment for a gene list against GO, KEGG, Reactome, WikiPathways, TRANSFAC,
miRTarBase, CORUM, HPA and HPO, plus gene/protein identifier conversion and cross-species ortholog
mapping — from g:Profiler at the University of Tartu.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1679+ live data sources.

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

Both URLs reach the same gateway and the same 1679+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/gprofiler_enrich \
  -H 'Content-Type: application/json' \
  -d '{"organism":"hsapiens","query":["TP53","BRCA1","BRCA2","ATM","CHEK2","PALB2"],"sources":["REAC"],"limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/gprofiler_enrich`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

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

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-gprofiler
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

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
