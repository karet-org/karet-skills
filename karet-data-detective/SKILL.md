---
name: karet-data-detective
description: |
  Diagnose why a Karet dashboard panel is empty, wrong, or showing
  stale data. Asks the user for their site URL (so it can point them
  at the right page) and walks the pipeline backwards through S3
  (panel → table → mapping → source → jobs) to find where rows are
  dropping out.
---

# karet-data-detective

## When to invoke

The user says the dashboard "looks wrong" or "is empty":

- "Why is my Total $0"
- "The line chart is flat"
- "Top Merchants is empty"
- "The category panel says 99% are OTHER"

## What you do

1. **Read the S3 environment.** `S3_BUCKET`, `AWS_ENDPOINT_URL` (only
   for RustFS / MinIO), `AWS_REGION`, plus the access keys. See
   `karet-dashboard-builder/references/s3-cheatsheet.md`.
2. **Ask for the site URL.** Get the URL or domain the user reaches
   their Karet site at (e.g. `http://localhost:3000`,
   `https://karet.mydomain.com`). You won't make HTTP calls to it --
   Karet has no public API -- but you'll quote URLs back so the user
   can verify your findings in the browser.
3. **Identify the suspect panel.** Pipeline slug, dashboard id, panel
   title. Pull the dashboard JSON from S3:

   ```sh
   aws s3 cp \
     "s3://$S3_BUCKET/pipelines/<slug>/dashboards/<id>.json" - \
     | jq '.panels[] | select(.title == "<title>")'
   ```

   Note which `column` / `agg` / `group_by` / `value` it uses, and
   which `analytic_table_id` the dashboard targets. Also read
   `filters[]` -- a too-narrow filter is the most common cause of
   "panel looks empty".
4. **Walk backwards through S3.** For each layer below, run the
   obvious sanity check:

   - **Panel** -- right column, right agg, right group_by. Is the
     filter knocking out all rows?
   - **Table** -- pull a recent parquet partition and read it:

     ```sh
     aws s3 cp \
       "s3://$S3_BUCKET/pipelines/<slug>/clean/<table>/year=YYYY/month=MM/data.parquet" \
       /tmp/sample.parquet
     duckdb -c "SELECT * FROM '/tmp/sample.parquet' LIMIT 50"
     ```

     Are the values what you'd expect? Is the column the panel uses
     all-null, all-zero, or empty?
   - **Mapping** -- read `pipeline.json`:

     ```sh
     aws s3 cp \
       "s3://$S3_BUCKET/pipelines/<slug>/pipeline.json" - \
       | jq '.mappings[] | select(.analytic_table_id == "<table>")'
     ```

     The expression for that column -- is it `cast` to `float64` from
     a string that has `$` or `,` in it? Is `parse_date` using the
     right chrono format?
   - **Lookup** -- if the panel groups by a lookup-derived column,
     check the corresponding `lookup_mappings[]` entry in
     `pipeline.json`. If most rows are landing in `catch_all` or
     `null`, the patterns are wrong.
   - **Source** -- list raw uploads and spot-check one:

     ```sh
     aws s3 ls "s3://$S3_BUCKET/pipelines/<slug>/raw/<container>/"
     aws s3 cp \
       "s3://$S3_BUCKET/pipelines/<slug>/raw/<container>/<file>.csv" - \
       | head -n 5
     ```

     Is there data there at all? Does the header match what the
     source container declares?
   - **Jobs** -- read the most recent run record:

     ```sh
     aws s3 ls "s3://$S3_BUCKET/pipelines/<slug>/jobs/" \
       | sort -r | head -n 5
     aws s3 cp \
       "s3://$S3_BUCKET/pipelines/<slug>/jobs/<id>.json" - \
       | jq '{status, startedAt, finishedAt, errors}'
     ```

     Did the latest run actually complete? How many partitions? Were
     there errors in the `errors[]` array?
5. **Report.** State the failure layer first, then the chain. See
   `references/findings-format.md`. Quote URLs at the user's site
   (the URL you collected in step 2) so they can verify -- e.g.
   "open `<site>/p/<slug>/jobs` to see the failed run".

## Hard rules

- **MUST** walk the actual data, not just the config. A static read
  of `pipeline.json` misses runtime issues like Polars failing to
  parse a `parse_date`.
- **MUST NOT** suggest a fix without showing the evidence. Quote
  numbers: "rows in raw: 4823. rows in clean: 4801. 22 dropped."
- **MUST** distinguish between *failures* (worker error in
  `jobs/<id>.json` `errors[]`) and *empty groups* (rows present but
  no longer match the filter or group_by).
- **MUST NOT** assume there's a Karet HTTP API. Every read is `aws s3
  cp` or `aws s3 ls`.
- **MUST NOT** read or modify `_auth/admin.json`. It's the admin
  password and unrelated to pipeline diagnostics.
- **MUST NOT** modify anything during diagnosis. Read-only walk only.
  If the conclusion calls for a config change, hand off to
  `karet-dashboard-tweaker` or `karet-pipeline-builder`.

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

Verify in the UI: <site>/p/monthly-spending/dashboards/monthly_overview
```
