# PipelineConfig JSON shape

Source of truth: `src/karet/lib/types/config.ts`. This file is a
copy-paste reference; the live types may have evolved.

```jsonc
{
  "version": 1,
  "source_containers": [
    {
      "id": "transactions_raw",
      "name": "Transactions",
      "path_prefix": "raw/transactions/",
      "schema": [
        { "name": "date",        "type": "string" },
        { "name": "description", "type": "string" },
        { "name": "amount",      "type": "string" },
        { "name": "account",     "type": "string" }
      ]
    }
  ],
  "lookup_mappings": [
    {
      "id": "categories",
      "name": "Categories",
      "match": "keyword_substring",
      "case_insensitive": true,
      "rows": [
        { "input_patterns": ["STARBUCKS","CAFE","RAMEN"], "output": "FOOD" },
        { "input_patterns": ["UBER","LYFT","SHELL"],      "output": "TRANSPORT" }
      ],
      "children": []
    }
  ],
  "mappings": [
    {
      "id": "transactions_mapping",
      "name": "Transactions Mapping",
      "source_container_id": "transactions_raw",
      "analytic_table_id": "transactions",
      "partition_by": { "column": "date", "granularity": "month" },
      "columns": [
        { "name": "date",        "expr": { "kind": "parse_date", "input": { "kind": "col", "name": "date" }, "format": "%Y-%m-%d" } },
        { "name": "description", "expr": { "kind": "upper", "input": { "kind": "col", "name": "description" } } },
        { "name": "amount",      "expr": { "kind": "cast",  "input": { "kind": "col", "name": "amount" }, "to": "float64" } },
        { "name": "account",     "expr": { "kind": "col",   "name": "account" } },
        { "name": "category",    "expr": { "kind": "lookup_ref", "lookup_id": "categories", "input": { "kind": "upper", "input": { "kind": "col", "name": "description" } } } }
      ]
    }
  ],
  "analytic_tables": [
    {
      "id": "transactions",
      "name": "Transactions",
      "output_prefix": "clean/transactions/",
      "schema": [
        { "name": "date",        "type": "date" },
        { "name": "description", "type": "string" },
        { "name": "amount",      "type": "float64" },
        { "name": "account",     "type": "string" },
        { "name": "category",    "type": "string" }
      ]
    }
  ]
}
```

## Notes

- `path_prefix` is relative to the pipeline's S3 prefix
  (`pipelines/<slug>/`). Don't add a leading slash.
- `output_prefix` is also relative; the worker writes
  `clean/<table>/year=YYYY/month=MM/data.parquet`.
- `partition_by.column` must be a column produced by the mapping (not
  a raw source column).
- `partition_by.granularity` is one of `day | week | month | year`.
- Source `schema` types are loose strings (`string | number | int64
  | float64 | date | bool`). Analytic table `schema` types are the
  canonical Karet set: `string | int64 | float64 | date | bool`.
- `lookup_mappings.children` is for nested lookups. Leave `[]` unless
  the user asks.
- `match` strategies: `exact`, `keyword_substring`, `regex`. Default
  to `keyword_substring` with `case_insensitive: true`.
