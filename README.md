# Idescat MCP server

MCP (Model Context Protocol) server that exposes tools to query the official
statistics of Catalonia, published by **Idescat** (Institut d'Estadística de
Catalunya), from any MCP client (Claude, Windsurf, Cursor, VS Code, etc.).

- **Public endpoint:** `https://api.idescat.cat/mcp`
- **Transport:** Streamable HTTP (MCP spec 2025-06-18), **stateless** — each POST request is independent, with no sessions. POST responses use JSON or SSE according to the client's `Accept` header; no persistent SSE stream is offered via GET.
- **Authentication:** none required (the data is public).
- **Data:** official Idescat statistics in **JSON-stat**; labels and descriptions are in **Catalan**; territorial scope is **Catalonia** and its territorial breakdowns (counties, municipalities, provinces, etc.).

## Installation

The public endpoint is `https://api.idescat.cat/mcp`. Add one of the snippets
below to your MCP client's configuration. Different clients expect a different
key (`serverUrl`, `url`, ...) and sometimes a `type`.

### Windsurf/Devin, Antigravity (`serverUrl`)

```json
{
	"mcpServers": {
		"idescat": {
			"serverUrl": "https://api.idescat.cat/mcp"
		}
	}
}
```

### Cursor, Roo Code, Zoo Code (`type: streamable-http`)

```json
{
	"mcpServers": {
		"idescat": {
			"type": "streamable-http",
			"url": "https://api.idescat.cat/mcp"
		}
	}
}
```

### Cline (`type: streamableHttp`)

```json
{
	"mcpServers": {
		"idescat": {
			"type": "streamableHttp",
			"url": "https://api.idescat.cat/mcp"
		}
	}
}
```

### VS Code and the emerging standard form (`servers`, `type: http`)

```json
{
	"servers": {
		"idescat": {
			"type": "http",
			"url": "https://api.idescat.cat/mcp"
		}
	}
}
```

### Claude.ai (custom connector)

Claude.ai (web and desktop apps) can reach the remote server directly as a
**custom connector**, no configuration file needed.

1. Open **Customize → Connectors** 
2. Click **Add custom connector**.
3. Fill in the fields:
   - **Name:** `Idescat`
   - **Remote MCP server URL:** `https://api.idescat.cat/mcp`
4. Click **Add**. No authentication (OAuth / API key) is needed, since the data
   is public.
5. Back in a chat, enable the **Idescat** connector from the tools/attachments
   menu. Its tools and resources are now available to the assistant.
6. (Optional) In the connector's tool settings, switch each Idescat tool from
   **Needs approval** to **Always allow** so the assistant can run them without
   asking for confirmation every time. All the tools are read-only, so this is
   safe.

### Claude Desktop and stdio-only clients (bridge with `mcp-remote`)

Clients that only speak stdio can reach the remote server through the
`mcp-remote` bridge:

```json
{
	"mcpServers": {
		"idescat": {
			"command": "npx",
			"args": ["-y", "mcp-remote", "https://api.idescat.cat/mcp"]
		}
	}
}
```

### Not sure which key your client expects? (catch-all)

Most clients ignore keys they don't recognise, so this superset works with many
of them at once:

```json
{
	"mcpServers": {
		"idescat": {
			"url": "https://api.idescat.cat/mcp",
			"serverUrl": "https://api.idescat.cat/mcp",
			"type": "streamable-http",
			"streamableHttp": "https://api.idescat.cat/mcp"
		}
	}
}
```

## Available tools

| Tool | Description |
| --- | --- |
| `get_idescat_stats` | Full list of available statistics. |
| `get_idescat_stat` | Information about a statistic (by acronym: `censph`, `pibc`, ...). |
| `get_idescat_stat_tables` | List of tables of a statistic. |
| `get_table_metadata` | Table metadata (dimensions, categories, units, ...). |
| `get_table_data` | Table data in JSON-stat: full, or filtered by dimension/territory/periods with the optional `filters` and `last` parameters (max. 20,000 cells after applying filters). Supports server-side calculations: age-range filters (`">=65"`), `sum` of a dimension's categories and `derive` (share / difference / percent_change). |
| `get_table_geo` | Territorial divisions available for a table (`cat`, `prov`, `com`, `mun`, ...). |
| `render_table` | Ready-pivoted Markdown table (selectable rows/columns) with an attribution link to the table on the Idescat website. Accepts the same calculations as `get_table_data` (age-range filters, `sum`, `derive`) and reports them in a "Calculations:" block under the table. |

## Available resources

Beyond the tools, the server exposes MCP **resources** — content a client can
attach to the context without executing any action.

Static resources (`resources/list`):

| URI | Description |
| --- | --- |
| `idescat://instructions` | Usage guidance. |
| `idescat://glossary/geo` | Glossary of territorial codes (`cat`, `prov`, `com`, `mun`, ...). |
| `idescat://catalog/stats` | Full catalogue of statistics. |

Resource templates (`resources/templates/list`) — parametrised URIs:

| URI template | Description |
| --- | --- |
| `idescat://stat/{stat}` | Information about a statistic (e.g. `.../censph`). |
| `idescat://stat/{stat}/tables` | List of tables of a statistic. |
| `idescat://table/{stat}/{table}` | Table metadata (e.g. `.../censph/538`). |
| `idescat://table/{stat}/{table}/data` | JSON-stat data of a table (max. 20,000 cells). |

> The transport is stateless, so `subscribe` and `resources/list_changed`
> notifications are **not** offered (there is no persistent connection).

## Server-side calculations

Language models are notoriously unreliable at mental arithmetic on long series
of values. `get_table_data` and `render_table` therefore accept three optional
parameters that make the server compute the result deterministically instead
of leaving the model to add, subtract or divide the returned values itself:

```json
{ "filters": { "AGE": ">=65", "SEX": "TOTAL" }, "last": 1 }
{ "sum": ["AGE"] }
{ "derive": { "share": "AGE", "percent_change": "previous" } }
```

- **Age-range filters.** For dimensions coded `Y000`, `Y005_009`, `Y_GE100`,
  etc., a filter value can be a range expression (`">=65"`, `">65"`, `"<16"`,
  `"<=15"`, `"16-64"`); the server resolves it to the exact list of codes and
  fails with `range_not_representable` when the published groups cannot express
  the bound exactly.
- **`sum`** collapses the listed dimensions by adding up their categories; the
  result is a single `SUM` category. `TOTAL` is excluded automatically.
- **`derive`** appends derived metrics to the metric dimension:
  `share` (`<METRIC>_SHARE` = 100 × value / value at `TOTAL` of the given
  dimension), `difference` (`<METRIC>_DIFF` vs. the previous or a given period)
  and `percent_change` (`<METRIC>_PCT` = 100 × (value − base) / base).

The engine relies on Idescat's conventions: additivity is declared by
`extension["%"]` on the metric dimension (only additive metrics can be summed
or shared), `TOTAL` is the total code, and non-computable cells come back as
`null` with status `d`. Everything applied is recorded in `extension.derived`
(ranges, summed codes, formulas, warnings) and `render_table` prints it under
the table as a `Calculations:` block.

Unsafe requests are refused: `metric_not_additive`, `geo_not_summable` (the
territorial dimension is never summed — use the `geo` parameter with the
division that already publishes the aggregate), `range_not_representable`,
`incomplete_data` and `total_not_available`.

## Agent skills

Two skills in the [Agent Skills](https://agentskills.io/) open format (a
directory with a `SKILL.md` file: YAML frontmatter + Markdown instructions)
teach agents how to use Idescat data correctly. They are published on the
website and advertised in the [ARD manifest](#discovery-agentic-resource-discovery):

| Skill | URL | Description |
| --- | --- | --- |
| `jsonstat` | <https://www.idescat.cat/dev/api/taules/jsonstat/SKILL.md> | How to read JSON-stat 2.0 datasets: dimension index, row-major order, stride/offset arithmetic, `null` and `status` handling. Generic — useful with any JSON-stat producer. |
| `workflow` | <https://www.idescat.cat/dev/api/taules/workflow/SKILL.md> | How to query Idescat data reliably (via this MCP server or the Tables API): choosing the statistic and territorial level, metadata-first flow, the 20,000-cell limit and filters, and mandatory source citation. |

## Repository files

The following files define the server's guidance and resources and are served,
unchanged, to MCP clients:

- **[`instructions.md`](instructions.md)** — usage guidance the server sends to
  clients during the handshake and exposes as the `idescat://instructions`
  resource. It explains how to choose a statistic, the data-retrieval flow, how
  to interpret JSON-stat, and the source-attribution rules.
- **[`resources.json`](resources.json)** — declaration of the MCP **resources**
  and **resource templates** the server exposes (the two tables in the
  [Available resources](#available-resources) section above), including each
  URI, title, description, MIME type, and its backing file or tool.
- **[`resources/geo.md`](resources/geo.md)** — glossary of territorial codes
  (`cat`, `prov`, `at`, `com`, `mun`, `dis`, `sec`) used by the `geo` parameter,
  exposed as the `idescat://glossary/geo` resource.

The repository also includes:

- **[`server.json`](server.json)** — the server's metadata manifest in the
  standard
  [server.json format](https://github.com/modelcontextprotocol/registry/blob/main/docs/reference/server-json/generic-server-json.md),
  used to publish the server to the official
  [MCP Registry](https://registry.modelcontextprotocol.io).
- **[`jsonstat/SKILL.md`](jsonstat/SKILL.md)** and
  **[`workflow/SKILL.md`](workflow/SKILL.md)** — the two
  [agent skills](#agent-skills), also published on the website under
  `https://www.idescat.cat/dev/api/taules/`.

## Discovery (Agentic Resource Discovery)

The server is advertised using the **Agentic Resource Discovery (ARD)**
specification (v0.91) so agents can find it automatically:

- An ARD manifest is published at
  `https://www.idescat.cat/.well-known/ard.json` and
  `https://api.idescat.cat/.well-known/ard.json` (served with
  `Access-Control-Allow-Origin: *` on both hosts). It carries a top-level
  `specVersion` so the same document is valid both as an ARD manifest and as
  an [AI Catalog](https://github.com/Agent-Card/ai-catalog) file:
  `/.well-known/ai-catalog.json` is served directly by an internal rewrite of
  `ard.json` (no redirect) as `application/ai-catalog+json` — the predecessor
  format that some MCP discovery flows still request first.
  The manifest advertises four entries: this MCP server
  (`application/mcp-server-card+json`), the
  [Idescat Tables API](https://www.idescat.cat/dev/api/taules/), described by
  its OpenAPI document (`application/openapi+json`) at
  `https://www.idescat.cat/dev/api/taules/openapi.json`, and the two
  [agent skills](#agent-skills)
  (`application/agent-skills+md`, the AI Catalog media type for a skill
  published as a Markdown `SKILL.md` file).
- The **MCP server card** (`application/mcp-server-card+json`) is served at
  `https://api.idescat.cat/mcp/server-card`, the location the experimental
  [Server Card discovery specification](https://github.com/modelcontextprotocol/ext-server-card/blob/main/docs/discovery.md)
  reserves as `<streamable-http-url>/server-card`. A `GET` (or `HEAD`) on the
  bare endpoint `https://api.idescat.cat/mcp` returns `405` — the endpoint is
  POST-only, although POST responses can use SSE. The Card and the MCP
  handshake (`serverInfo`) share the name `cat.idescat/mcp`, matching the
  MCP Registry. The Card response honours the spec's caching contract:
  `Cache-Control: public, max-age=3600`, an `ETag` that answers
  `If-None-Match` with `304`, and CORS headers (`Allow-Methods: GET`,
  `Allow-Headers: Content-Type, If-None-Match`, `Expose-Headers: ETag`).
- The legacy Card aliases `/.well-known/mcp/server-card.json` and
  `/.well-known/mcp.json` are served directly on `api.idescat.cat` by an
  internal rewrite (identical bytes, no redirect — some crawlers do not
  follow them); on `www.idescat.cat` they redirect (307) to
  `https://api.idescat.cat/mcp/server-card`.
- The MCP entry in `ard.json` points to the Card URL and exposes the
  transport endpoint via `metadata.mcpEndpoint`
  (`https://api.idescat.cat/mcp`). It uses `application/mcp-server-card+json`:
  the older `application/mcp-server+json` media type was deprecated in
  [ARD ADR-0008](https://github.com/ards-project/ard-spec/blob/main/adr/0008-mcp-media-type-renaming.md).
- The website pages also include a link to the manifest:
  `<link rel="ard" href="/.well-known/ard.json" type="application/json">`
  (the `ard` link relation replaces the predecessor `ai-catalog` relation).
- `robots.txt` includes an `Agentmap:` directive pointing to the manifest.
- The MCP server is also advertised in the site's `llms.txt` file.

### MCP Registry

The server is listed in the official
[MCP Registry](https://registry.modelcontextprotocol.io) under the name
`cat.idescat/mcp` (see [`server.json`](server.json)). MCP clients and
aggregators can retrieve its metadata from the registry API:

    https://registry.modelcontextprotocol.io/v0.1/servers/cat.idescat%2Fmcp/versions/latest


## Interactive testing with MCP Inspector

```bash
npx @modelcontextprotocol/inspector@2.0.0
```

Pinning a specific version (e.g. `@2.0.0`) keeps the tool reproducible and
avoids picking up unexpected changes; it requires Node.js **>= 22.19.0**. Any
`npm warn deprecated ...` / `ERESOLVE` messages come from the Inspector's own
dependencies and are harmless — they do not affect this MCP server.

Select the "Streamable HTTP" transport and enter the endpoint URL.

## Links

- Idescat: <https://www.idescat.cat>
- Model Context Protocol: <https://modelcontextprotocol.io>
- MCP specification (2025-06-18, Streamable HTTP transport): <https://modelcontextprotocol.io/specification/2025-06-18>
- Agentic Resource Discovery (ARD): <https://agenticresourcediscovery.org>
- ARD specification (rendered): <https://agenticresourcediscovery.org/spec/>
- ARD specification repository (schemas, conformance tooling): <https://github.com/ards-project/ard-spec>
