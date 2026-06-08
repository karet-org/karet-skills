# Mapping JSON shape

```jsonc
{
  "id": "transactions_mapping",
  "name": "Transactions Mapping",
  "source_container_id": "transactions_raw",
  "analytic_table_id": "transactions",
  "partition_by": { "column": "date", "granularity": "month" },
  "columns": [
    { "name": "<target-col-name>", "expr": <ast-node> }
  ]
}
```

See `karet-pipeline-builder/references/ast-cookbook.md` for every
`AstNode` primitive and recipe.

## Translation table by target type

| target type | source `string` → wrap in… | notes |
|------------|----------------------------|-------|
| `string` | `{ kind: "col", name }` | Add `upper`/`lower`/`trim` if downstream lookups need it. |
| `int64` | `{ kind: "cast", input: <col>, to: "int64" }` | Wrap in `trim` if values have whitespace. |
| `float64` | `{ kind: "cast", input: <col>, to: "float64" }` | Wrap in `trim` if values have whitespace. Currency formatting (`$1,234.56`) needs an additional substring/replace pass; ask the user. |
| `date` | `{ kind: "parse_date", input: <col>, format: "..." }` | Always confirm the chrono format string. |
| `bool` | `{ kind: "cast", input: <col>, to: "bool" }` | Polars accepts `"true"/"false"`, `"1"/"0"`, `"yes"/"no"`. |

## Partitioning

`partition_by.column` must be a column produced by the mapping (i.e.
in the target schema), not a raw source column. Only `month`
granularity is implemented; any other value fails the run with
`UnsupportedGranularity`.

## Lookup-derived columns

There are two common shapes.

**Strict tagging** (`category`-style): use a `lookup_ref` directly.
Unmatched rows return `null` (or the lookup's `catch_all` if set). Use
this when the lookup is a closed enum (e.g. ISO country codes) and a
miss should be visibly null.

```jsonc
{ "name": "category",
  "expr": { "kind": "lookup_ref", "lookup_id": "categories",
            "input": <upper(trim(col(description)))> } }
```

**Normalize-with-fallback** (`merchant`-style): wrap the lookup in
`coalesce` paired with the cleaned input. Use this when the lookup is
a curated subset of a long tail (e.g. canonical merchant names) and
unmatched rows should still appear in the dashboard with their raw
description.

```jsonc
{ "name": "merchant",
  "expr": { "kind": "coalesce",
            "args": [
              { "kind": "lookup_ref", "lookup_id": "merchants",
                "input": <upper(trim(col(description)))> },
              <upper(trim(col(description)))>
            ] } }
```
