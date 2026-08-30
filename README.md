# Idescat MCP server

MCP (Model Context Protocol) server that exposes tools to query the official
statistics of Catalonia, published by **Idescat** (Institut d'Estadística de
Catalunya), from any MCP client (Claude, Windsurf, Cursor, VS Code, etc.).

- **Public endpoint:** `https://api.idescat.cat/mcp`
- **Transport:** Streamable HTTP (MCP spec 2025-06-18), **stateless** — each POST request is independent, with no sessions or SSE streams.
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
| `get_table_data` | Table data in JSON-stat: full, or filtered by dimension/territory/periods with the optional `filters` and `last` parameters (max. 20,000 cells after applying filters). |
| `get_table_geo` | Territorial divisions available for a table (`cat`, `prov`, `com`, `mun`, ...). |
| `render_table` | Ready-pivoted Markdown table (selectable rows/columns) with an attribution link to the table on the Idescat website. |

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
- **[`server.json`](server.json)** — the server's metadata manifest in the
  standard
  [server.json format](https://github.com/modelcontextprotocol/registry/blob/main/docs/reference/server-json/generic-server-json.md),
  used to publish the server to the official
  [MCP Registry](https://registry.modelcontextprotocol.io).

## Discovery (Agentic Resource Discovery)

The server is advertised using the **Agentic Resource Discovery (ARD)**
specification (v0.91) so agents can find it automatically:

- An ARD manifest is published at
  `https://www.idescat.cat/.well-known/ard.json` and
  `https://api.idescat.cat/.well-known/ard.json`. The legacy predecessor path
  (`/.well-known/ai-catalog.json`) is redirected (301) to the new manifest.
  The manifest advertises two entries: this MCP server
  (`application/mcp-server-card+json`) and the
  [Idescat Tables API](https://www.idescat.cat/dev/api/taules/), described by
  its OpenAPI document (`application/openapi+json`) at
  `https://www.idescat.cat/dev/api/taules/openapi.json`.
- A `GET` request to the endpoint `https://api.idescat.cat/mcp` returns the
  **MCP server card** (`application/mcp-server-card+json`), generated from the
  server's tool and resource definitions. A `GET` requesting
  `text/event-stream` returns `405`, since this stateless server offers no SSE
  stream.
- The MCP server card is also served at
  `https://www.idescat.cat/.well-known/mcp/server-card.json` (and `https://www.idescat.cat/.well-known/mcp.json`), which redirects
  to `https://api.idescat.cat/mcp`.
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

    https://registry.modelcontextprotocol.io/v0.1/servers/cat.idescat%2Fmcp


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
