---
name: karet-data-detective
description: |
  Diagnose why a dashboard panel is empty, wrong, or showing stale
  data. Walks the pipeline backwards (panel → table → mapping →
  source) and reports where rows are dropping out.
---

# karet-data-detective

## When to invoke

The user says the dashboard "looks wrong" or "is empty":

- "Why is my Total $0"
- "The line chart is flat"
- "Top Merchants is empty"
- "The category panel says 99% are OTHER"

## What you do

1. **Identify the suspect panel.** Pipeline slug, dashboard id, panel
   title. Read the dashboard JSON to see which columns and aggregation
   that panel uses.
2. **Walk backwards.** For each layer, run the obvious sanity check:
   - **Panel** -- right column, right agg, right group_by. Is the
     filter knocking out all rows? Read the dashboard's `filters[]`.
   - **Table** -- `GET /api/p/<slug>/tables/<table>/rows?limit=50`.
     Are the values what you'd expect? Is the column the panel uses
     all-null, all-zero, or empty?
   - **Mapping** -- read `pipeline.json`. The expression for that
     column -- is it `cast` to `float64` from a string that has
     `$` or `,` in it? Is `parse_date` using the right chrono format?
   - **Lookup** -- if the panel groups by a lookup-derived column,
     check the lookup. If most rows are landing in `catch_all` or
     `null`, the patterns are wrong.
   - **Source** -- list raw files at
     `pipelines/<slug>/raw/...`. Is there data there at all? Does the
     header match what the source container declares?
   - **Jobs** -- `GET /api/p/<slug>/jobs`. Did the latest run
     actually complete? How many partitions? Were there errors in
     the `errors[]` array?
3. **Report.** State the failure layer first, then the chain. See
   `references/findings-format.md`.

## Hard rules

- **MUST** walk the actual data, not just the config. A static read
  of the JSON misses runtime issues like Polars failing to parse a
  `parse_date`.
- **MUST NOT** suggest a fix without showing the evidence. Quote
  numbers: "rows in raw: 4823. rows in clean: 4801. 22 dropped."
- **MUST** distinguish between *failures* (worker error in `errors[]`)
  and *empty groups* (rows present but no longer match the filter
  or group_by).

## Output

```
Diagnosed `monthly-spending` / `monthly_overview` / "Top Merchants":

  PANEL    expects column `description`, agg `sum`, value `amount`
  FILTERS  account dropdown is set to "amex-gold" (only 4% of rows)
  TABLE    `transactions` has 4823 rows, 187 distinct descriptions
  MAPPING  description column is `upper(col(description))` -- looks fine
  JOBS     last run completed 8 min ago, 4823 rows, 0 errors

Most likely cause: the filter. Clear the account dropdown to see all
rows. With the filter cleared, "Top Merchants" should show RENT,
PG&E, COMCAST, TARGET, SHELL.
```
