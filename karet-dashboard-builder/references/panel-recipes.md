# Panel recipes

When you read a table schema, fall back on these defaults.

## A "spending"-shape table (date, description, amount, account, category)

```
KPIs:    Total ($SUM amount), Count, Top category
Charts:  Doughnut by category, Line by month, Top-5 bar by description
Table:   All columns, paginated
Filters: Account dropdown, Date range
```

## A sensor/IoT-shape table (timestamp, host, metric, value)

```
KPIs:    Avg, Max, Distinct host count
Charts:  Line by day (avg value), Doughnut by host, Top-5 bar by metric
Table:   timestamp, host, metric, value
Filters: Host dropdown, Date range
```

## A geographic table (lat/lon or country)

```
KPIs:    1-2 headline numbers (count + max value)
Charts:  Symbol map (lat/lon), or Choropleth (country),
         + supporting Top-N bar
Table:   First 5 columns, paginated
Filters: Date range only -- geo is its own filter
```

## A "narrow" table (2-3 columns)

```
KPIs:    Sum, Count
Charts:  Line if there's a date, otherwise Top-N bar
Table:   Full columns
Filters: None
```

## Sizing rules of thumb

- Three KPIs on one row: each takes `auto` width, the layout's
  `gridTemplateColumns` handles wrapping.
- Doughnut + line side by side: doughnut gets `aspect: square,
  maxHeight: 20rem`, line gets `gridColumn: span 2`.
- Top-N bar: full row, `gridColumn: 1 / -1`.
- Tables: always full row, always last.
- Maps: `gridColumn: span 2` and `aspect: video`.

## Don't do

- KPI on a date column (the `mode` aggregation will return a
  semi-random timestamp).
- Doughnut with `> 12` distinct categories. Use a top-N bar instead.
- Line chart on data without a clear date column.
- More than 7 panels on a single dashboard. Split into two if you
  need more.
- `agg: "sum"` on a string column (no error, but always 0).
