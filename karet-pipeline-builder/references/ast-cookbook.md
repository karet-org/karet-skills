# AST cookbook

The mapping column expression language is a discriminated union.
Source of truth: `AstNode` in `src/karet/lib/types/config.ts`.

## Atoms

```jsonc
{ "kind": "col",  "name": "description" }   // read a source column
{ "kind": "str",  "value": "USD" }          // literal string
{ "kind": "num",  "value": 1.5 }            // literal number
{ "kind": "bool", "value": true }           // literal bool
{ "kind": "null" }                          // literal null
```

## Type coercion

```jsonc
{ "kind": "cast", "input": { "kind": "col", "name": "amount" }, "to": "float64" }
// `to`: int64 | float64 | string | date
```

## Strings

```jsonc
{ "kind": "upper", "input": <expr> }
{ "kind": "lower", "input": <expr> }
{ "kind": "trim",  "input": <expr> }

{ "kind": "substring", "input": <expr>, "start": 0, "length": 4 }   // length null -> to end
{ "kind": "concat",    "sep": " ",      "args": [<expr>, <expr>] }
```

## Dates

```jsonc
{ "kind": "parse_date", "input": <expr>, "format": "%Y-%m-%d" }
// chrono format. Common patterns:
//   "%Y-%m-%d"        2026-04-15
//   "%m/%d/%Y"        04/15/2026
//   "%d-%b-%Y"        15-Apr-2026
//   "%Y-%m-%dT%H:%M"  2026-04-15T13:45
```

## Comparisons / booleans

```jsonc
{ "kind": "eq", "left": <expr>, "right": <expr> }   // also: ne gt lt ge le
{ "kind": "contains", "input": <expr>, "pattern": <expr> }
```

## Conditionals

```jsonc
{
  "kind": "if",
  "cond": { "kind": "lt", "left": { "kind": "col", "name": "amount" }, "right": { "kind": "num", "value": 0 } },
  "then": { "kind": "str", "value": "REFUND" },
  "else": { "kind": "str", "value": "CHARGE" }
}
```

## Lookups

```jsonc
{
  "kind": "lookup_ref",
  "lookup_id": "categories",
  "input": { "kind": "upper", "input": { "kind": "col", "name": "description" } }
}
```

The `input` is what gets matched against the lookup's `input_patterns`.
Always normalize before lookup if the lookup is `case_insensitive`.
The worker does case-fold itself, but matching is more predictable
when both sides agree.

When the lookup has no `catch_all`, an unmatched input returns `null`.
Pair with `coalesce` to fall back to the raw input. See the
"Normalize-with-fallback" recipe below.

## Coalesce

```jsonc
{ "kind": "coalesce", "args": [<expr1>, <expr2>, …] }
```

Returns the first non-null arg; `null` if every arg is null. Typical
use: wrap a `lookup_ref` against a curated subset and supply the raw
input as the fallback arg.

## Arithmetic

```jsonc
{ "kind": "add", "left": <expr>, "right": <expr> }   // also: sub mul div
```

## Common recipes

**Parse currency string into float**:

```jsonc
{ "kind": "cast",
  "input": { "kind": "trim", "input": { "kind": "col", "name": "amount" } },
  "to": "float64" }
```

**Year-month bucket from a date column** (only useful inside expressions
that the partitioner doesn't already handle for you):

```jsonc
{ "kind": "substring",
  "input": { "kind": "col", "name": "iso_date" },
  "start": 0, "length": 7 }
```

**Normalize-with-fallback** (canonical merchant name when matched, raw
cleaned description otherwise):

```jsonc
{ "kind": "coalesce",
  "args": [
    { "kind": "lookup_ref",
      "lookup_id": "merchants",
      "input": { "kind": "upper",
                 "input": { "kind": "trim",
                            "input": { "kind": "col", "name": "description" } } } },
    { "kind": "upper",
      "input": { "kind": "trim",
                 "input": { "kind": "col", "name": "description" } } }
  ] }
```

**Conditionally negate** (turn a positive amount into a signed one based
on a debit/credit flag):

```jsonc
{ "kind": "if",
  "cond": { "kind": "eq",
            "left": { "kind": "upper", "input": { "kind": "col", "name": "type" } },
            "right": { "kind": "str", "value": "DEBIT" } },
  "then": { "kind": "mul",
            "left": { "kind": "cast", "input": { "kind": "col", "name": "amount" }, "to": "float64" },
            "right": { "kind": "num", "value": -1 } },
  "else": { "kind": "cast", "input": { "kind": "col", "name": "amount" }, "to": "float64" } }
```
