---
name: karet-pipeline-importer
description: |
  Given a directory of CSVs, scaffold a Karet pipeline that has one
  Source per CSV pattern, a draft Mapping per Source, and an Analytic
  Table for each. Writes pipeline.json and uploads the CSVs straight
  to S3 -- the bucket webhook auto-runs the pipeline once data lands.
  For users with a folder of monthly exports who want to wire it all
  up in one go.
---

# karet-pipeline-importer

## When to invoke

The user has a folder of CSV files following a naming pattern:

- "Wire up everything in `~/exports/`"
- "I have a year of monthly transaction CSVs"
- "Set up a pipeline for my Strava activity exports in this folder"

## What you do

1. **Read the S3 environment.** `S3_BUCKET`, `AWS_ENDPOINT_URL` (only
   if pointing at RustFS / MinIO; unset for real AWS), `AWS_REGION`,
   `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`. See
   `karet-pipeline-builder/references/s3-cheatsheet.md`.
2. **List the directory.** Look for CSVs, group by stem pattern.
   Common shapes:
   - All same shape, time-stamped: `transactions-2026-01.csv`,
     `transactions-2026-02.csv`, ... → one source, one mapping, one
     table.
   - Multiple types: `transactions-*.csv`, `accounts-*.csv` → one
     source per type.
3. **Pick a slug.** Ask the user. Default from the directory name.
   Confirm the slug isn't already taken with `aws s3 ls
   s3://$S3_BUCKET/pipelines/<slug>/pipeline.json`.
4. **For each group**, profile the CSV (delegate to
   `karet-csv-inspector`). Get a column-by-column type table.
5. **Build the pipeline JSON.** One source container with the
   appropriate `path_prefix`, one mapping that does sensible parses
   (delegate to `karet-mapping-author` per group), one analytic
   table. Validate locally (see step 7 of
   `karet-pipeline-builder/SKILL.md`).
6. **Write the pipeline config.**

   ```sh
   aws s3 cp pipeline.json \
     "s3://$S3_BUCKET/pipelines/<slug>/pipeline.json" \
     --content-type application/json
   ```

7. **Upload the CSVs.** `aws s3 cp` (or `aws s3 sync` for a whole
   directory) into
   `s3://$S3_BUCKET/pipelines/<slug>/raw/<container-prefix>/`.
   Each `ObjectCreated` event fires the bucket webhook, which the web
   app debounces into a single run. There's no manual trigger needed.

   ```sh
   aws s3 sync ./exports/transactions \
     "s3://$S3_BUCKET/pipelines/<slug>/raw/transactions/" \
     --exclude '*' --include '*.csv'
   ```

8. **Confirm.** Ask the user for the URL / domain their Karet site is
   at if you don't know it yet. Print the slug, the file count
   queued, and a link to the jobs page (`<site>/p/<slug>/jobs`).

## Hard rules

- **MUST** profile every distinct CSV shape. Two files with the same
  name pattern but different headers should each get their own
  source.
- **MUST NOT** assume all CSVs in the folder belong to one pipeline.
  Confirm with the user if more than one shape is detected.
- **MUST** write `pipeline.json` *before* uploading the CSVs.
  Otherwise the webhook fires against a missing config and the worker
  will mark the run failed.
- **MUST NOT** call any Karet HTTP API. There isn't one for managing
  pipelines -- writes go to S3, runs are scheduled by the bucket
  webhook.
- **MUST NOT** read or write `_auth/admin.json` or anything under
  `clean/` (worker output, read-only).
- **MUST** include the user's directory name as the pipeline `name`
  default, but let them override.

## Output

```
Imported pipeline `bank-exports`:
  3 source containers (transactions, accounts, balances)
  3 mappings, 3 analytic tables
  47 CSVs uploaded to s3://$S3_BUCKET/pipelines/bank-exports/raw/

The bucket webhook will debounce these uploads into a single run.
Watch progress at <site>/p/bank-exports/jobs
```
