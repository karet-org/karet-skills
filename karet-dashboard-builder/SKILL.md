---
name: karet-dashboard-builder
description: |
  Generate a Karet dashboard for an existing Analytic Table. Reads the
  pipeline config and a parquet sample directly from S3, asks the user
  a few targeted questions, and writes the DashboardConfig JSON back
  to S3 at pipelines/<slug>/dashboards/<id>.json.
---

# karet-dashboard-builder

## When to invoke

The user has a populated Analytic Table and wants charts. They might
say:

- "Make me a dashboard for the `transactions` table"
- "I want to see monthly spending and top merchants"
- "Build a status dashboard for the `inverter_logs` pipeline"

## What you do

1. **Read the S3 environment.** `S3_BUCKET`, `AWS_ENDPOINT_URL` (only
   for RustFS / MinIO -- unset for real AWS), `AWS_REGION`, plus
   `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`. See
   `references/s3-cheatsheet.md`.
2. **Resolve the target.** Ask for the pipeline slug if not given.
   Check it exists: `aws s3 ls
   s3://$S3_BUCKET/pipelines/<slug>/pipeline.json`.
3. **Read the schema.** Pull the pipeline config straight from S3:

   ```sh
   aws s3 cp "s3://$S3_BUCKET/pipelines/<slug>/pipeline.json" -
   ```

   Find the user's Analytic Table in `analytic_tables[]` to learn
   column names and types. If the user named a table that doesn't
   exist, list the available `analytic_tables[].id`s and ask.
4. **Sample real data** (recommended). Pull one parquet partition out
   of `clean/<table>/` and read it with whatever's handy
   (`duckdb -c "SELECT * FROM 'sample.parquet' LIMIT 20"`,
   `parquet-tools head`, or a Polars one-liner). Helps you pick
   sensible KPIs (skip a `currency`-formatted KPI on a column whose
   values are all `0`) and bin granularities (skip `month` for
   hourly sensor data). If `clean/` is empty, the table hasn't been
   populated yet -- tell the user and proceed without samples.
5. **Interview.** Ask:
   - What's the headline number for this dataset? (becomes a KPI)
   - What dimensions do you want to slice by?
   - Is there a time series worth charting?
   - Do any geo columns exist (lat/lon, country code)?
6. **Pick a panel set.** See `references/panel-recipes.md` for the
   common compositions. Default to: 3 KPIs + 1 doughnut + 1 line +
   1 top-N bar + 1 transactions table. Don't overload -- 5-7 panels
   is the sweet spot.
7. **Pick filters.** `dropdown` on the lowest-cardinality
   string column (typically a category or account). `date_range` if
   the table has a `date` column.
8. **Compose the JSON.** See `references/dashboard-shape.md`.
9. **Write it to S3.** The renderer reads dashboards from
   `s3://$S3_BUCKET/pipelines/<slug>/dashboards/<id>.json` on every
   page load -- the file appearing is the deploy step. See
   `references/s3-cheatsheet.md` for the `aws s3 cp` recipe.
10. **Confirm.** Ask the user for the URL/domain their Karet site is
    at if you don't know it yet, and point them at
    `<site>/p/<slug>/dashboards/<id>`.

## Hard rules

- **MUST NOT** reference columns that aren't in the target Analytic
  Table schema. The dashboard renderer silently shows zero or "missing
  column" errors.
- **MUST** match column types to panel kinds. `line.x` should be a
  `date` column; `kpi.column` for `agg: "sum"` must be numeric;
  `choropleth_map.country` must be a string.
- **MUST NOT** invent dashboard IDs. Use a slug derived from the
  dashboard name (lowercase, underscored). The S3 key uses this id
  verbatim and the URL routes by it.
- **MUST NOT** read or write `_auth/admin.json` or touch anything
  under `clean/` (read-only worker output) beyond pulling sample
  parquet files.
- **MUST NOT** assume a Karet HTTP API exists for dashboards.
  Reads and writes both go through S3.

## Style guidance

- Three KPI tiles across the top, then chart panels below.
- Doughnut for category breakdowns (5-10 categories max).
- Top-N bar (`limit: 5`) for high-cardinality dimensions like
  merchants or hostnames.
- Line for time series; bin to `month` for human data, `day` for
  ops/IoT data.
- Tables go full width (`grid: { gridColumn: "1 / -1" }`) at the
  bottom, with a `page_size` of 8-12.

## Output

```
Created dashboard `monthly_overview` on pipeline `monthly-spending`.
Wrote s3://$S3_BUCKET/pipelines/monthly-spending/dashboards/monthly_overview.json.
Open <site>/p/monthly-spending/dashboards/monthly_overview to verify.
```
