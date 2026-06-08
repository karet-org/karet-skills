# LookupMapping JSON shape

```jsonc
{
  "id": "categories",
  "name": "Categories",
  "match": "keyword_substring",
  "case_insensitive": true,
  "rows": [
    { "input_patterns": ["STARBUCKS", "CAFE", "RAMEN"], "output": "FOOD" },
    { "input_patterns": ["UBER", "LYFT", "SHELL"],      "output": "TRANSPORT" }
  ],
  "children": [],
  "catch_all": { "output": "OTHER" }
}
```

## Fields

- **`id`**: referenced by `lookup_ref` AstNodes elsewhere. Keep short
  and lowercase.
- **`match`**: write `"keyword_substring"`. The worker matches by
  case-(in)sensitive **substring** only and does not read this field
  today, so `exact` and `regex` do NOT exist -- setting them still
  yields substring behavior. Don't offer other strategies.
- **`case_insensitive`**: defaults to `false`. Almost always set to
  `true` for human-typed data.
- **`rows`**: `{ input_patterns: string[], output: string, priority?: number }[]`.
- **`priority`** (per row, optional): tie-breaker when more than one
  row matches the same input. The matcher picks the matching row with
  the highest `priority`; ties fall back to definition order. Defaults
  to `0`, so omitting it preserves first-match-wins. Use it when a
  broad pattern would otherwise shadow a specific one (e.g. an `AMAZON`
  → SHOPPING row and an `AMAZON ... PAYROLL` → INCOME row: give INCOME
  a higher priority).
- **`children`**: nested lookups for hierarchical category structures.
  Leave `[]` unless the user explicitly asks.
- **`catch_all`**: optional. `{ output: string }` returned when no
  pattern matches. Without it, no-match rows get a `null` output.

## Use site

Reference the lookup from a mapping column expression:

```jsonc
{ "name": "category",
  "expr": {
    "kind": "lookup_ref",
    "lookup_id": "categories",
    "input": { "kind": "upper", "input": { "kind": "col", "name": "description" } }
  } }
```

Always normalize the `input` (upper/lower/trim) so the match is
predictable, especially when `case_insensitive: true`.

## When to skip `catch_all`

For "normalize-with-fallback" lookups (e.g. merchant canonicalization
where unmatched descriptions should keep showing up by their raw form
rather than collapsing into a single "OTHER" bucket), leave `catch_all`
unset and pair the lookup with `coalesce` at the use site:

```jsonc
{ "name": "merchant",
  "expr": { "kind": "coalesce",
            "args": [
              { "kind": "lookup_ref", "lookup_id": "merchants", "input": <cleaned> },
              <cleaned>
            ] } }
```

Without `catch_all`, the `lookup_ref` returns `null` on a miss and the
`coalesce` falls back to the cleaned input. The Spending Tracker
template's `merchant` column uses this shape.
