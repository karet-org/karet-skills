# DashboardConfig JSON shape

Source of truth: `src/karet/lib/types/dashboard.ts`. Excerpted here
for offline reference.

```jsonc
{
  "id": "monthly_overview",
  "name": "Monthly Overview",
  "analytic_table_id": "transactions",

  "filters": [
    { "kind": "dropdown",   "column": "account",  "label": "Account" },
    { "kind": "date_range", "column": "date",     "label": "Date range" }
  ],

  "panels": [
    {
      "kind": "kpi",
      "title": "Total Spending",
      "column": "amount",
      "agg": "sum",
      "format": "currency",
      "currency": "CAD",
      "icon": "dollar"
    },
    {
      "kind": "kpi",
      "title": "Transactions",
      "column": "amount",
      "agg": "count",
      "format": "number",
      "icon": "chart"
    },
    {
      "kind": "kpi",
      "title": "Top Category",
      "column": "category",
      "agg": "mode",
      "value_column": "amount",
      "format": "currency",
      "currency": "CAD",
      "icon": "shapes"
    },

    {
      "kind": "doughnut",
      "title": "By Category",
      "group_by": "category",
      "value": "amount",
      "agg": "sum",
      "grid": { "aspect": "square", "maxHeight": "20rem" }
    },
    {
      "kind": "line",
      "title": "Monthly Trend",
      "x": "date",
      "x_bin": "month",
      "y": "amount",
      "agg": "sum",
      "grid": { "gridColumn": "span 2" }
    },
    {
      "kind": "bar",
      "title": "Top Merchants",
      "group_by": "description",
      "value": "amount",
      "agg": "sum",
      "limit": 5,
      "grid": { "gridColumn": "1 / -1" }
    },
    {
      "kind": "table",
      "title": "Transactions",
      "columns": ["date","description","amount","account","category"],
      "page_size": 8,
      "grid": { "gridColumn": "1 / -1" }
    }
  ],

  "layout": {
    "gridTemplateColumns": "repeat(auto-fit, minmax(max(18rem, calc((100% - 2rem) / 3)), 1fr))",
    "gap": "1rem"
  }
}
```

## Notes

- `id` doubles as the URL slug
  (`/p/<pipeline>/dashboards/<id>`) and as the S3 key
  (`pipelines/<pipeline>/dashboards/<id>.json`). Keep it
  filesystem-safe: lowercase + underscore.
- `analytic_table_id` must match an Analytic Table on the same
  pipeline.
- `filters[].column` must be a column on the target table.
- `agg` values: `sum | count | avg | min | max`. KPIs additionally
  support `mode` (most common value, paired with `value_column` for
  the displayed total).
- `format` on KPIs: `number | currency | raw`. Without `currency` the
  default is USD.
- `icon` on KPIs: `dollar | chart | shapes | calendar`.
- `x_bin` on line charts: `day | week | month | year`.
- `grid.gridColumn` uses CSS grid syntax. `"span 2"` makes a panel
  twice as wide; `"1 / -1"` spans the full row.
- `grid.aspect`: `"square"` for circles/doughnuts, `"video"` for maps,
  `"auto"` (default) for everything else. `maxHeight` clamps the
  chart height -- always set it for square panels so they don't blow
  up on wide screens.
