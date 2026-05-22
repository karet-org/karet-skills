---
name: karet-csv-inspector
description: |
  Profile a CSV file -- column types, null rate, cardinality, candidate
  keys -- and emit a ready-to-paste source-container schema and a
  starter mapping. Use when the user has a spreadsheet and doesn't
  know what types to declare.
---

# karet-csv-inspector

## When to invoke

The user has a CSV (path, URL, or pasted snippet) and is asking
questions like "what types should I use", "is this column unique",
"what should I cast", or just "look at this file". Often invoked at
the start of a pipeline-build flow before
`karet-pipeline-builder` takes over.

## What you do

1. **Read the data.** Local path, URL, or pasted text -- whichever the
   user has. Read at least the header + 100 sampled rows. Larger
   files: stream the first 10k rows.
2. **Profile per column.**
   - **Detected type**: try in order: `int64`, `float64`, `date`
     (parse with chrono format candidates -- `%Y-%m-%d`, `%m/%d/%Y`,
     `%d-%b-%Y`, `%Y-%m-%dT%H:%M`), `bool`, fall through to `string`.
   - **Null rate**: count empty / `NULL` / `N/A` cases.
   - **Cardinality**: distinct values. Mark "low cardinality" if
     `< 20` distinct values across `> 100` rows -- those are dropdown
     filter candidates.
   - **Sample values**: 3 representative non-null values.
3. **Detect candidate keys.** Columns with `100%` distinct values and
   `0%` null rate. Flag them as candidate primary keys.
4. **Detect lookup-target columns.** Low-cardinality string columns
   are candidates for `lookup_ref` enrichment (categorize, normalize).
5. **Emit two artifacts.** A profile report (markdown table) and a
   source-container `schema` block ready to paste into a `pipeline.json`.
   See `references/profile-report.md` for the format.
6. **Suggest follow-ups.** If a date column was detected, mention the
   chrono format string. If a numeric column has currency formatting
   (`$1,234.56`, `(45.67)`), warn about the `cast` recipe.

## Hard rules

- **MUST NOT** auto-correct data. Report what you saw, including
  ambiguous cases ("78% of values look like dates, the other 22% are
  blank or `N/A`").
- **MUST** state which date format string you matched, and on what
  fraction of rows. Karet's `parse_date` is strict.
- **MUST NOT** guess types from column *names* -- only from observed
  values. A column called `id` may still hold strings; a column called
  `total_amount` may be `string` if the source pads with currency.

## Output

```
Inspected `transactions.csv`. 4823 rows, 6 columns.

| column      | type    | null% | distinct | sample values            | notes                       |
|-------------|---------|-------|----------|--------------------------|-----------------------------|
| date        | date    | 0%    | 365      | 2026-04-15, 2026-05-02   | format: %Y-%m-%d (100%)     |
| description | string  | 0%    | 1247     | STARBUCKS, RENT, UBER    | high cardinality            |
| amount      | string  | 0%    | 4081     | "1,234.56", "45.67"      | needs trim+cast (commas)    |
| account     | string  | 0%    | 3        | visa-1234, amex-gold     | low cardinality, dropdown   |
| memo        | string  | 87%   | 142      | "tax", "tip", null       | mostly null, skip?          |
| ...

Suggested source-container schema:
[paste]

Caveats: `amount` has thousands separators; cast won't work without
a trim+replace pre-step. `memo` is 87% null -- consider dropping.
```
