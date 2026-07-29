# Territorial code glossary (`geo`)

Codes accepted by the `geo` parameter of the tools (`get_table_metadata`, `get_table_data`, `get_table_geo`, `render_table`) and by the statistics' `geo` field.

| Code | Description |
| --- | --- |
| `cat` | Whole of Catalonia (default) |
| `prov` | Provinces |
| `at` | Territorial plan areas |
| `com` | Counties (Comarques) and Aran |
| `mun` | Municipalities |
| `dis` | Districts |
| `sec` | Census tracts |

**Notes:**

- The default value is `cat` if not specified.
- Not every table offers all divisions: check `link.related` in `get_table_metadata` or `get_table_geo`.
- Divisions other than `cat` (especially `mun`, `dis`, `sec`) often exceed the 20,000-cell limit; use `get_table_data` with filters.
