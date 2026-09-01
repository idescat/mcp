---
name: workflow
description: Query the official statistics of Catalonia (Idescat) correctly, either through the Idescat MCP server or the Idescat Tables REST API. Use when retrieving Catalan statistical data (population, GDP, employment, etc.) to choose the right statistic and territorial level, follow the metadata-first retrieval flow, respect the 20,000-cell limit with dimension filters, and cite sources as required.
license: CC BY 4.0
metadata:
  author: idescat
  version: "1.0.0"
---

# Idescat data workflow

How to query the official statistics of Catalonia (Idescat) reliably. It
applies both to the **Idescat MCP server** (`https://api.idescat.cat/mcp`)
and to the **Idescat Tables REST API**
(`https://api.idescat.cat/taules/v2`, OpenAPI description at
`https://www.idescat.cat/dev/api/taules/openapi.json`). Responses follow
the JSON-stat format (see the companion `jsonstat` skill for how to read
them).

## 1. Choosing a statistic

Always start by listing the available statistics (MCP tool
`get_idescat_stats`; API `GET /taules/v2`). For each statistic, check two
key fields before choosing:

- `datasets` (boolean): if `true` the statistic is normalized and you can
  drill down to tables and data. If `false`, only the general information
  about the statistic is available.
- `geo` (array): available territorial breakdown levels (`cat` =
  Catalonia, `prov` = provinces, `at` = territorial plan areas, `com` =
  comarques and Aran, `mun` = municipalities, `dis` = districts, `sec` =
  census tracts). ALWAYS check `geo` before choosing a statistic for a
  territorial query: if comarca-level data is requested, pick a statistic
  whose `geo` includes `com`.

### Population figures

When population data is requested there are several sources: the main and
richest one is **CENSPH** (population and housing census, annual); for
half-yearly estimates there is **EP** (population estimates, half-yearly);
and there is also **PMH** (municipal population register, annual). By
default use CENSPH (or EP for half-yearly data). NEVER use the PMH
statistic for population data unless the user explicitly asks for it.

## 2. Data-retrieval flow

1. When specific data from a table is needed, fetch the table METADATA
   first (MCP `get_table_metadata`; API
   `GET /taules/v2/{statistics}/{node}/{table}/{geo}`). If the question is
   about a specific territory, pass the proper `geo` (e.g. `com`, `mun`,
   `prov`); if you don't know the territorial divisions available, list
   them first (MCP `get_table_geo`; API
   `GET /taules/v2/{statistics}/{node}/{table}`). Remember that without a
   territorial breakdown the response is the Catalonia total (`cat`),
   which does NOT include comarques, municipalities or provinces.
2. Check the metadata `size` field: multiply all its elements. If the
   product exceeds 20,000, do not fetch the whole table (the API rejects
   it with HTTP 416); filter first, or tell the user and offer the HTML
   link to the table.
3. Fetch the data (MCP `get_table_data` or `render_table`; API
   `GET /taules/v2/{statistics}/{node}/{table}/{geo}/data`) with optional
   filters:
   - For a SINGLE territory or a subset, filter on the territorial
     dimension (e.g. `geo` = `com` and filter `COM=21`), and/or limit the
     time periods with `_LAST_` (e.g. `_LAST_=2` for the last two
     periods).
   - Dimension filters are query parameters named after the dimension
     identifiers shown in the metadata, with comma-separated category
     codes as values (e.g. `?SEX=F&COM=01,TOTAL`).
   - When you need the full dataset (≤ 20,000 cells), fetch it without
     filters.

## 3. Data truthfulness and source (MANDATORY)

- NEVER use internal knowledge or training data to give figures, dates,
  amounts, percentages, rankings or any statistical value. Every reported
  value must come LITERALLY from a call made in the same conversation.
- If no call was made, the call returned an error, or the dataset is too
  large, do NOT invent or infer data. Clearly state the data could not be
  obtained and offer the table link.
- Whenever data is cited it is MANDATORY to include the Markdown link to
  the source table. Use the `source` link when the response provides one;
  otherwise build it as `https://www.idescat.cat/pub/?id={stat}&n={table}`.
  NEVER present data without this source link.
- If in doubt about the origin of a figure, do not give it.

## 4. Presentation conventions

- Statistic acronyms: write them in UPPERCASE in responses (e.g. EPA,
  AFI, CENSPH, PIBA), but pass them in lowercase as request parameters
  (e.g. `epa`, `afi`, `censph`, `piba`).
- Tables have a numeric identifier: refer to it as "identifier" or
  "table [number]", never as "node_id", "node" or any other internal
  technical denomination.
