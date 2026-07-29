# Idescat MCP — Usage guidance

Tools to query the official statistics of Catalonia (Idescat): list of statistics, tables and data in JSON-stat format.

---

## 1. Choosing a statistic

Always start with `get_idescat_stats` to locate the statistic. For each one, check two key fields before choosing:
- `datasets` (boolean): if `true` the statistic is normalized and you can use the rest of the tools (`get_idescat_stat_tables`, `get_table_metadata`, `get_table_data`, `get_table_geo`, `render_table`). If `false`, ONLY `get_idescat_stats` and `get_idescat_stat` are available.
- `geo` (array): available territorial breakdown levels (`cat`=Catalonia, `prov`=provinces, `at`=territorial plan areas, `com`=comarques and Aran, `mun`=municipalities, `dis`=districts, `sec`=census tracts). ALWAYS check `geo` before choosing a statistic for a territorial query: if comarca-level data is requested, pick a statistic whose `geo` includes `com`.

### Population figures

When population data is requested there are several sources: the main and richest one is **CENSPH** (population and housing census, annual); for half-yearly estimates there is **EP** (population estimates, half-yearly); and there is also **PMH** (municipal population register, annual). By default use CENSPH (or EP for half-yearly data). NEVER use the PMH statistic for population data unless the user explicitly asks for it.

---

## 2. Data-retrieval flow

1. When the user asks for specific data from a table, call `get_table_metadata` first. If the question is about a specific territory (comarca, municipality, province, area), pass the proper `geo` parameter (e.g. `geo='com'`, `geo='mun'`, `geo='prov'`); if you don't know the codes, call `get_table_geo`.
2. Check the metadata `size` field: multiply all array elements. If the product exceeds 20,000, do not fetch the whole table; tell the user and offer the HTML link to the table.
3. To fetch data always use `get_table_data` (or `render_table`), which accepts optional filters:
   - For a SINGLE territory or a subset (e.g. GDP of one comarca), call `get_table_data` with the proper `geo` and a filter on the territorial dimension (e.g. `geo='com'` and `filters={"COM":"21"}`), and/or `last` to limit the periods. Remember that without `geo` the call returns the Catalonia total (`geo='cat'`) by default, which does NOT include comarques, municipalities or provinces.
   - When you need the full dataset (≤ 20,000 cells), call `get_table_data` without filters.

---

## 3. Interpreting JSON-stat

Note: within a JSON-stat dataset, `id` refers to the ORDERED list of dimension names (see below) — this is unrelated to a statistic's acronym (`stat` parameter, section 5) or a table's numeric identifier (section 6).

Dataset structure:
- `id`: ORDERED list of dimension names. This is the order that governs position calculations (NOT the key order of `dimension`).
- `size`: number of categories per dimension (same order as `id`).
- `dimension.{name}.category.index`: 0-based position of each category code within the dimension. It comes in one of two forms: an object `{code: position}` or a list `[code0, code1, ...]` (the position is the list index). ALWAYS use this index; do not assume alphabetical or label order.
- `dimension.{name}.category.label`: code → human-readable label.
- `value`: FLAT array with all values, in row-major order ([link](https://en.wikipedia.org/wiki/Row-major_order)): the LAST `id` dimension varies fastest and the first varies slowest.
- `status`: flags special values (e.g. provisional or confidential data).

To READ the value of a specific combination of categories (one per dimension):
1. For each dimension, take the position `p` of the chosen category from `category.index`.
2. Compute each dimension's stride = product of the `size` of ALL later dimensions; the last dimension has stride = 1.
3. `offset` = sum of (`p` × stride) over all dimensions.
4. The value is `value[offset]`.

Example: `id=['territory','sex']`, `size=[3,2]`; `territory.index={BCN:0,GIR:1,LLE:2}`, `sex.index={M:0,F:1}` → strides `[2,1]`. For Girona+Female: `offset = 1×2 + 1×1 = 3` → `value[3]`. Canonical sample: https://json-stat.org/samples/order.json

Notes:
- `null` (data not available) OCCUPIES its slot in the order: it does NOT shift the index.
- "Base 2024", "Base 2024=100" or "2024 Statistical Revision" in the title do NOT mean estimates; they only indicate the methodological reference year (data is real unless `status` says otherwise).

---

## 4. Data truthfulness and source (MANDATORY)

- NEVER use your internal knowledge or training data to give figures, dates, amounts, percentages, rankings or any statistical value. Every reported value must come LITERALLY from a tool call made in this same conversation.
- If you haven't called a tool, the tool returned an error, or the dataset is too large, do NOT invent or infer data. Clearly state you couldn't obtain it and offer the table link.
- Whenever you cite data it is MANDATORY to include the Markdown link to the source table. Data tools return a `source` field with this link: use it. If you don't have it, build it as `https://www.idescat.cat/pub/?id={stat}&n={table}`. NEVER present data without this source link.
- If in doubt about the origin of a figure, do not give it.

---

## 5. Statistic acronyms: presentation and arguments

When mentioning a statistic's acronym in your responses, always write it in UPPERCASE (for example, EPA, AFI, CENSPH, PIBA). In contrast, when passing the acronym as a tool argument (the `stat` parameter), always write it in lowercase (e.g. `epa`, `afi`, `censph`, `piba`), as the parameters require.

---

## 6. Table naming

When presenting lists of tables, each table has a numeric identifier. Always refer to it as "identifier" or "table [number]", never as "node_id", "node" or any other internal technical denomination.

---

## 7. MCP resources

Besides the tools, the server exposes "resources" (context-attachable content, no action executed): `idescat://instructions` (this guide), `idescat://glossary/geo` (territorial codes), `idescat://catalog/stats` (statistics catalog) and templates like `idescat://stat/{stat}`, `idescat://stat/{stat}/tables`, `idescat://table/{stat}/{table}` and `idescat://table/{stat}/{table}/data`. These URIs are stable, citable references, but do NOT replace the mandatory Markdown link to the Idescat web table (section 4) when reporting data.
