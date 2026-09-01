---
name: jsonstat
description: Read and interpret JSON-stat 2.0 datasets correctly. Use when working with JSON-stat responses from statistical APIs (Idescat, Eurostat, national statistical offices) to locate a cell value in the flat row-major value array from its category combination, using the dimension index and stride/offset arithmetic, and to handle null values, status flags and collections.
license: CC BY 4.0
metadata:
  author: idescat
  version: "1.0.0"
---

# Reading JSON-stat datasets

[JSON-stat](https://json-stat.org/) is a JSON format for statistical data
used by official statistics providers (Idescat, Eurostat, national
statistical institutes). Its key feature — and the most common source of
mistakes — is that all cell values live in a single **flat array** whose
order is determined by the dataset's dimensions. Follow this guide to read
values reliably.

## Dataset structure

A JSON-stat **dataset** (`"class": "dataset"`) has these essential
properties:

- `id`: ORDERED list of dimension names. This order governs all position
  calculations (NOT the key order of the `dimension` object, which is
  irrelevant).
- `size`: number of categories per dimension, in the same order as `id`.
- `dimension.{name}.category.index`: 0-based position of each category code
  within the dimension. It comes in one of two forms:
  - an object `{code: position}`, e.g. `{"M": 0, "F": 1}`, or
  - a list `[code0, code1, ...]` (the position is the list index).
  ALWAYS use this index; never assume alphabetical or label order. If
  `category.index` is absent and `category.label` has a single key, the
  dimension has one category at position 0.
- `dimension.{name}.category.label`: map from category code to
  human-readable label.
- `value`: FLAT array with all cell values in
  [row-major order](https://en.wikipedia.org/wiki/Row-major_order): the
  LAST dimension of `id` varies fastest and the first varies slowest.
- `status`: flags special values (e.g. provisional or confidential data).
  It can be a string (applies to all values), an array (parallel to
  `value`) or an object keyed by value position.

## Reading the value of a category combination

To read the value for a specific combination of categories (one per
dimension):

1. For each dimension, take the position `p` of the chosen category from
   `category.index`.
2. Compute each dimension's **stride** = product of the `size` of ALL later
   dimensions (in `id` order); the last dimension has stride = 1.
3. `offset` = sum of (`p` × stride) over all dimensions.
4. The value is `value[offset]`.

### Worked example

`id = ["territory", "sex"]`, `size = [3, 2]`;
`territory.index = {"BCN": 0, "GIR": 1, "LLE": 2}`,
`sex.index = {"M": 0, "F": 1}`.

Strides: territory = 2 (size of `sex`), sex = 1.

For Girona + Female: `offset = 1×2 + 1×1 = 3` → the answer is `value[3]`.

Canonical sample to test against: https://json-stat.org/samples/order.json

## Pitfalls

- `null` in `value` means "data not available", but it OCCUPIES its slot:
  it does NOT shift the positions of the other values.
- Check `status` before presenting a figure: it marks provisional,
  estimated or confidential values. Dataset-level or dimension-level
  `extension.status` objects often explain each status symbol.
- "Base 2024", "Base 2024=100" or "2024 Statistical Revision" in a title
  do NOT mean the data is an estimate; they only indicate the
  methodological reference year (data is real unless `status` says
  otherwise).
- Do not confuse `id` (the ordered dimension list) with any
  provider-specific identifiers such as a statistic's acronym or a table
  number.

## Collections

A JSON-stat **collection** (`"class": "collection"`) is not data: it is a
list of links to other resources (datasets or further collections) under
`link.item[]`, each with `class`, `href` and `label`. Follow `href` to
reach the actual dataset.
