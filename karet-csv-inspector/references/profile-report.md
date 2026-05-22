# Profile report format

The profile report has three parts in this order:

## 1. Header

```
Inspected `<filename>`. <N> rows, <M> columns.
```

## 2. Per-column table

```
| column | type | null% | distinct | sample values | notes |
```

- **type**: detected canonical Karet type (`string | int64 | float64 |
  date | bool`), not the raw source-container type.
- **null%**: rounded to whole percent.
- **distinct**: count of unique non-null values, or `~N` if approximate
  (e.g. only sampled).
- **sample values**: 3 short non-null values, comma-separated.
- **notes**: free-form. Useful flags:
  - `format: %Y-%m-%d (100%)` -- chrono format and match rate
  - `low cardinality, dropdown` -- candidate filter column
  - `candidate key` -- 100% distinct, 0% null
  - `needs trim+cast` -- numeric-looking but has whitespace or
    formatting
  - `mostly null, skip?` -- > 50% null rate
  - `mixed dates: 78% %Y-%m-%d, 22% %m/%d/%Y` -- ambiguous

## 3. Source-container schema block

```jsonc
{
  "schema": [
    { "name": "date",        "type": "string" },
    { "name": "description", "type": "string" },
    { "name": "amount",      "type": "string" },
    ...
  ]
}
```

Note: source-container `type` should usually stay `string` even when
the column is numeric. Karet expects to parse and cast inside the
mapping. The exception is when the source is already a Parquet file
or another structured format that ships true types.

## 4. Caveats

A short bulleted list of things the user should know before wiring the
mapping:

- formatting issues that need trim/replace before `cast`
- mostly-null columns to consider dropping
- ambiguous date formats that need user confirmation
- candidate keys (mention so the user can use them in lookups)
