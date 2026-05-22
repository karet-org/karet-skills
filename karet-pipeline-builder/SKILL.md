---
name: karet-pipeline-builder
description: |
  Build a Karet pipeline from a description of the user's CSV data.
  Conducts a short interview, picks types, generates AST-JSON mapping
  expressions, and writes a complete pipeline.json into the user's
  Karet instance via the web API.
---

# karet-pipeline-builder

## When to invoke

The user wants to ingest CSV data into Karet but doesn't want to
hand-write the `pipeline.json`. They might say things like:

- "I have a folder of bank statements and want to chart spending"
- "I export Strava activities monthly. Set up a pipeline."
- "Help me wire up these IoT sensor logs in Karet"

## What you do

1. **Discover the running instance.** Read `KARET_BASE_URL` (default
   `http://localhost:3000`) and `KARET_API_KEY` from the user's env. If
   either is missing, ask. The API key comes from the `KARET_API_KEY`
   line in their `.env`.
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
5. **Build the config in memory.** See `references/pipeline-shape.md`
   for the exact JSON shape and `references/ast-cookbook.md` for the
   AST primitives you can compose.
6. **Validate locally before sending.** Every `Mapping.columns[i].name`
   must appear in the corresponding `AnalyticTable.schema`, every
   `mapping.source_container_id` must exist, and every `lookup_ref`
   must point at a `LookupMapping.id` you defined.
7. **POST to the Karet API.** See `references/api-cheatsheet.md`. Use
   `POST /api/pipelines` with `{ slug, template: "blank" }` to create
   the shell, then `PUT /api/p/<slug>/config` with the body you built.
   Honor the `If-Match` ETag.
8. **Confirm.** Open `<KARET_BASE_URL>/p/<slug>/graph` in the user's
   browser and report the slug back.

## Hard rules

- **MUST NOT** invent CSV columns. Only use names you saw in the
  sampled data or that the user explicitly named.
- **MUST NOT** call the worker (`http://worker:8080`) directly. Talk to
  the web API only.
- **MUST NOT** overwrite an existing pipeline. If the slug is taken,
  ask the user for a new one.
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

## Output

When done, print a one-line summary like:

> Created pipeline `monthly-spending` with 1 source, 1 lookup, 1
> mapping, and 1 analytic table. Open
> http://localhost:3000/p/monthly-spending/graph to verify and run.
