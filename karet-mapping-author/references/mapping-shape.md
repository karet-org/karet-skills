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
in the target schema), not a raw source column. Granularities:
`day | week | month | year`.
