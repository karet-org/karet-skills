---
name: karet-pipeline-builder
description: |
  Build a Karet pipeline from a description of the user's CSV data.
  Conducts a short interview, picks types, generates AST-JSON mapping
  expressions, and writes a complete pipeline.json directly into the
  user's S3 bucket. No HTTP API -- Karet reads pipeline.json from S3
  on every request.
---

# karet-pipeline-builder

## When to invoke

The user wants to ingest CSV data into Karet but doesn't want to
hand-write the `pipeline.json`. They might say things like:

- "I have a folder of bank statements and want to chart spending"
- "I export Strava activities monthly. Set up a pipeline."
- "Help me wire up these IoT sensor logs in Karet"

## What you do

1. **Read the S3 environment.** Pull `S3_BUCKET`, `AWS_ENDPOINT_URL`,
   `AWS_REGION`, and the access keys from the user's env (or the
   `.env` they start the stack with). If `S3_BUCKET` is missing,
   ask -- it defaults to `karet-data` but a self-hoster may have
   renamed it. See `references/s3-cheatsheet.md` for what's required.
2. **Sample the data.** If the user has a CSV path, read its header +
   first 20 rows. If they only have a description, ask for one
   representative row. Never invent column shapes you haven't seen.
3. **Interview.** Ask, in order:
   - What is this dataset about? (one sentence -- becomes the pipeline
     slug)
   - What's the primary timestamp column?
   - Are there categorical columns that would benefit from a lookup
     (categorize, normalize, map IDs to labels)?
   - What's the natural partition grain (`day` / `week` / `month` /
     `year`)? Default to `month` for human-readable data.
4. **Pick a slug.** Run the user-facing name through a slug-friendly
   transform (lowercase, hyphens). Confirm.
5. **Check that the slug is free.** `aws s3 ls
   s3://$S3_BUCKET/pipelines/<slug>/pipeline.json` -- if it exists,
   stop and ask the user for a different slug or for explicit
   permission to overwrite.
6. **Build the config in memory.** See `references/pipeline-shape.md`
   for the exact JSON shape and `references/ast-cookbook.md` for the
   AST primitives you can compose.
7. **Validate locally before writing.** Karet has no public validation
   endpoint -- the worker validates a config when it runs, but by
   then the bad JSON is already live. Check yourself:
   - JSON parses.
   - Every `Mapping.columns[i].name` appears in the corresponding
     `AnalyticTable.schema`.
   - Every `mapping.source_container_id` matches a
     `source_containers[].id`.
   - Every `lookup_ref.lookup_id` matches a `LookupMapping.id`.
   - `partition_by.column` is in the target table's schema.
8. **Write `pipeline.json` directly to S3.** See
   `references/s3-cheatsheet.md`. The path is
   `s3://$S3_BUCKET/pipelines/<slug>/pipeline.json` with
   `--content-type application/json`.
9. **Tell the user where to verify.** Ask for the URL/domain their
   Karet site is reachable at if you don't know it yet, then point
   them at `<site>/p/<slug>/graph`. Mention that they still need to
   upload CSVs into `pipelines/<slug>/raw/<container-prefix>/`
   before a run will produce data -- the bucket webhook fires on
   each upload and debounces a job.

## Hard rules

- **MUST NOT** invent CSV columns. Only use names you saw in the
  sampled data or that the user explicitly named.
- **MUST NOT** call the worker (`http://worker:8080`) directly. Write
  to S3 only.
- **MUST NOT** assume there's a Karet HTTP API for config. There
  isn't one. The web app and worker both read `pipeline.json`
  straight from S3.
- **MUST NOT** overwrite an existing `pipeline.json` unless the user
  explicitly asks. The S3 PUT is unconditional -- there's no `If-Match`
  to fall back on -- so the responsibility is yours.
- **MUST NOT** read or write `_auth/admin.json`. That's the admin
  password file; it's scrypt-hashed and unrelated to pipelines.
- **MUST** keep `Mapping.columns` in the same order as
  `AnalyticTable.schema`. Karet doesn't enforce this, but downstream
  dashboards read columns by index in some panels.

## Common pitfalls

- **Date parsing**: Polars chrono format strings, not Python's. `%Y` is
  4-digit year, `%m` is 2-digit month. Don't mix in `%d-%m-%Y` if the
  data is `2026-04-15` -- that's `%Y-%m-%d`.
- **Currency/amount columns**: source data is almost always `string`;
  cast with `{ kind: "cast", input: ..., to: "float64" }` after
  trimming. Negative-with-parens (`(45.67)`) won't parse, ask the user
  to confirm signing convention.
- **Lookups**: prefer `keyword_substring` with `case_insensitive: true`
  over an exact-match join unless the data is already canonicalised.
  Real-world descriptions rarely match exactly.
- **Partition column**: must be the column named in
  `AnalyticTable.schema`, after the mapping is applied -- not the raw
  source column. Once you `parse_date` a string into a date, partition
  on the date column.
- **Run trigger**: writing `pipeline.json` does *not* start a run.
  Runs fire when CSVs land in `pipelines/<slug>/raw/...`. Tell the
  user to upload data (or use `karet-pipeline-importer` to do both
  at once).

## Output

When done, print a one-line summary like:

> Created pipeline `monthly-spending` (1 source, 1 lookup, 1 mapping,
> 1 analytic table). Open `<site>/p/monthly-spending/graph` to verify;
> drop CSVs into `s3://$S3_BUCKET/pipelines/monthly-spending/raw/transactions/`
> to trigger the first run.
