---
name: karet-pipeline-importer
description: |
  Given a directory of CSVs, scaffold a Karet pipeline that has one
  Source per CSV pattern, a draft Mapping per Source, and an Analytic
  Table for each. For users with a folder of monthly exports who want
  to wire it all up in one go.
---

# karet-pipeline-importer

## When to invoke

The user has a folder of CSV files following a naming pattern:

- "Wire up everything in `~/exports/`"
- "I have a year of monthly transaction CSVs"
- "Set up a pipeline for my Strava activity exports in this folder"

## What you do

1. **List the directory.** Look for CSVs, group by stem pattern.
   Common shapes:
   - All same shape, time-stamped: `transactions-2026-01.csv`,
     `transactions-2026-02.csv`, ... → one source, one mapping, one
     table.
   - Multiple types: `transactions-*.csv`, `accounts-*.csv` → one
     source per type.
2. **Pick a slug.** Ask the user. Default from the directory name.
3. **For each group**, profile the CSV (delegate to
   `karet-csv-inspector`). Get a column-by-column type table.
4. **Build the pipeline JSON.** One source container with the
   appropriate `path_prefix`, one mapping that does sensible parses
   (delegate to `karet-mapping-author` per group), one analytic
   table.
5. **Create the pipeline.** `POST /api/pipelines` with `template:
   "blank"`, then `PUT /api/p/<slug>/config` with the assembled JSON.
6. **Upload the CSVs.** `aws s3 cp` each file into
   `pipelines/<slug>/raw/<container-prefix>/`. The webhook will
   schedule a run automatically once they land.
7. **Confirm.** Print the slug and a count of files queued for
   ingestion. Suggest opening the jobs page to watch progress.

## Hard rules

- **MUST** profile every distinct CSV shape. Two files with the
  same name pattern but different headers should each get their own
  source.
- **MUST NOT** assume all CSVs in the folder belong to one pipeline.
  Confirm with the user if more than one shape is detected.
- **MUST** include the user's directory name as the pipeline `name`
  default, but let them override.

## Output

```
Imported pipeline `bank-exports`:
  3 source containers (transactions, accounts, balances)
  3 mappings, 3 analytic tables
  47 CSVs uploaded to S3

Run progress: http://localhost:3000/p/bank-exports/jobs
```
