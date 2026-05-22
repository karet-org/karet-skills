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
2. **Pick a `match` strategy.**
   - **`keyword_substring`** -- the input pattern is found anywhere in
     the lookup target. Default for category-style data where source
     descriptions are noisy ("STARBUCKS #4582" matches "STARBUCKS").
   - **`exact`** -- the entire lookup target equals the input pattern.
     Use when both sides are canonical (ISO country codes, internal
     account IDs).
   - **`regex`** -- the lookup target matches the regex. Use sparingly
     -- harder to debug.
3. **Pick a case strategy.** Default `case_insensitive: true`. Real
   data is rarely case-stable.
4. **Group inputs by output.** Each `LookupRow` is `{ input_patterns:
   string[], output: string }`. Multiple inputs per output collapse
   into one row.
5. **Emit the JSON.** See `references/lookup-shape.md`.
6. **(Optional) Catch-all.** Ask if there should be a default output
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
