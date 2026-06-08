---
name: karet-lookup-builder
description: |
  Build a Karet LookupMapping from a CSV of input/output pairs. Use
  for category mappings, ISO country codes, account-to-team rosters,
  or any "many inputs map to one output" join.
---

# karet-lookup-builder

## When to invoke

The user has a list of patterns and outputs they want to use to
enrich a column. Common shapes:

- "I have a list of merchants and the categories they belong to"
- "Here are our internal account IDs mapped to team names"
- "Map ISO-3166 alpha-2 codes to country names"

## What you do

1. **Read the CSV / paste.** Two-column shape: `input` (or
   `input_patterns`), `output`. If the user has many inputs per output,
   the input column may be pipe-delimited (`STARBUCKS|CAFE|RAMEN`)
   or the file may have one row per pattern with repeated outputs.
2. **Set the `match` field.** The worker's matcher does
   case-(in)sensitive **substring** matching only: a row matches when any
   `input_pattern` is found anywhere in the input (`STARBUCKS #4582`
   matches `STARBUCKS`). Write `match: "keyword_substring"` for clarity,
   but be aware the worker does not actually read this field today, so
   `exact` and `regex` are NOT implemented. Do not offer them; a config
   that sets `match: "exact"` still gets substring behavior, silently.
   If you need exact matching, anchor it yourself (e.g. canonicalize both
   sides upstream) rather than relying on a `match` value.
3. **Pick a case strategy.** Default `case_insensitive: true`. Real
   data is rarely case-stable.
4. **Group inputs by output.** Each `LookupRow` is `{ input_patterns:
   string[], output: string }`. Multiple inputs per output collapse
   into one row.
5. **Resolve overlaps with `priority`.** Since matching is substring
   based, a broad pattern can shadow a specific one (e.g. `AMAZON` →
   SHOPPING would catch `AMAZON ... PAYROLL` before an INCOME row). When
   two rows can match the same input, set a higher `priority` (integer,
   default `0`) on the row that should win. Ties fall back to definition
   order.
6. **Emit the JSON.** See `references/lookup-shape.md`.
7. **(Optional) Catch-all.** Ask if there should be a default output
   when no input matches. If yes, include `catch_all: { output: "..." }`.

## Hard rules

- **MUST** dedupe `input_patterns`. Duplicate entries don't break
  anything but make the file noisy.
- **MUST** warn if any pattern is empty or whitespace-only -- those
  match every row and break the lookup.
- **MUST NOT** silently lowercase or normalize patterns. Show the
  user what you'll write and ask if they want it normalized.

## Output

```
Generated LookupMapping `categories`:
  match: keyword_substring (case-insensitive)
  rows: 7 outputs / 32 input patterns
  catch_all: OTHER

Add to your pipeline.json:
[paste]
```
