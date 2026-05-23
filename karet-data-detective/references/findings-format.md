# Findings format

## Header

```
Diagnosed `<slug>` / `<dashboard>` / "<panel-title>":
```

## Layer chain

For each layer, one line. Format:

```
  LAYER  <observation>
```

Layers in order: `PANEL`, `FILTERS`, `TABLE`, `MAPPING`, `LOOKUP`,
`SOURCE`, `JOBS`. Skip layers that are not relevant.

Numbers are mandatory:

- "4823 rows in raw, 4801 in clean -- 22 dropped"
- "187 distinct descriptions"
- "filter set to `amex-gold` matches 4% of rows"
- "last run: 8 min ago, 0 errors"

## Conclusion

```
Most likely cause: <one sentence>
<one to three sentences explaining what to do>
```

## Common diagnoses

- **Filter eliminates all rows** -- the dropdown or date range is
  excluding everything. Easy to confirm: clear the filter, the
  panel populates.
- **`parse_date` failures** -- mapping uses `%Y-%m-%d` but data is
  `MM/DD/YYYY`. Check `jobs.errors[]` for parse errors. Drops happen
  silently into the partition we couldn't determine.
- **Lookup catch-all everything** -- panel grouped by a lookup
  column, but `input_patterns` don't match real descriptions. Read
  raw `description` values and pick canonical patterns.
- **Empty / stale partitions** -- the job didn't run after the last
  CSV upload. List `pipelines/<slug>/jobs/` in S3; if the most recent
  job is older than the most recent CSV under
  `pipelines/<slug>/raw/`, the webhook didn't fire (check
  `KARET_WEBHOOK_SECRET` is set on both sides) or the run is still
  debouncing. The user can click "Run" on `<site>/p/<slug>/jobs`
  to trigger one manually.
- **Wrong agg** -- `sum` on a string column always returns 0.
  Always confirm the column type matches the agg.
- **Worker not running** -- if `jobs.errors[0]` is a network /
  timeout error, check that the worker container is up.
