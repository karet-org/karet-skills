---
name: karet-dashboard-builder
description: |
  Generate a Karet dashboard for an existing Analytic Table. Inspects
  the table's schema, asks the user a few targeted questions, and
  writes a DashboardConfig JSON to S3 via the Karet web API.
---

# karet-dashboard-builder

## When to invoke

The user has a populated Analytic Table and wants charts. They might
say:

- "Make me a dashboard for the `transactions` table"
- "I want to see monthly spending and top merchants"
- "Build a status dashboard for the `inverter_logs` pipeline"

## What you do

1. **Resolve the target.** Ask for the pipeline slug if not given.
   Read `KARET_BASE_URL` and `KARET_API_KEY` from env. If either is
   missing, ask.
2. **Read the schema.** `GET /api/p/<slug>/config`. Read the target
   `analytic_tables[].schema` to learn column names and types. If the
   user named a table that doesn't exist, list available ones and ask.
3. **Sample real data** (recommended). `GET /api/p/<slug>/tables/<table>/rows?limit=20`.
   Helps you pick sensible KPIs (skip a `currency`-formatted KPI on a
   column whose values are all `0`) and bin granularities (skip
   `month` for hourly sensor data).
4. **Interview.** Ask:
   - What's the headline number for this dataset? (becomes a KPI)
   - What dimensions do you want to slice by?
   - Is there a time series worth charting?
   - Do any geo columns exist (lat/lon, country code)?
5. **Pick a panel set.** See `references/panel-recipes.md` for the
   common compositions. Default to: 3 KPIs + 1 doughnut + 1 line +
   1 top-N bar + 1 transactions table. Don't overload -- 5-7 panels
   is the sweet spot.
6. **Pick filters.** `dropdown` on the lowest-cardinality
   string column (typically a category or account). `date_range` if
   the table has a `date` column.
7. **Compose the JSON.** See `references/dashboard-shape.md`.
8. **Write it.** Karet has no PUT route for dashboards. The skill
   writes the JSON directly to S3 at
   `pipelines/<slug>/dashboards/<dashboard-id>.json`. See
   `references/api-cheatsheet.md` for the env vars and the `aws s3 cp`
   call.
9. **Confirm.** Open
   `<KARET_BASE_URL>/p/<slug>/dashboards/<dashboard-id>` and report
   back.

## Hard rules

- **MUST NOT** reference columns that aren't in the target Analytic
  Table schema. The dashboard renderer silently shows zero or "missing
  column" errors.
- **MUST** match column types to panel kinds. `line.x` should be a
  `date` column; `kpi.column` for `agg: "sum"` must be numeric;
  `choropleth_map.country` must be a string.
- **MUST NOT** invent dashboard IDs. Use a slug derived from the
  dashboard name (lowercase, underscored).

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
http://localhost:3000/p/monthly-spending/dashboards/monthly_overview
```
