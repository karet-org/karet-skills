# Karet pipeline S3 cheatsheet

Karet has no HTTP API for managing pipelines. Both the web app and
the worker read config straight from S3, so the skill writes there
directly with `aws s3`.

## Required environment

The same vars Karet itself reads -- find them in the stack's `.env`:

| Variable | Notes |
|----------|-------|
| `S3_BUCKET` | Bucket name. Default `karet-data`. |
| `AWS_ENDPOINT_URL` | S3 endpoint. e.g. `http://localhost:9000` for local RustFS. **Unset** for real AWS S3. |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | S3 credentials. |
| `AWS_REGION` | Required even for RustFS. Use `us-east-1` if you don't care. |

The CLI picks `AWS_ENDPOINT_URL` up automatically (since AWS CLI v2.13).
On older CLI versions, pass `--endpoint-url "$AWS_ENDPOINT_URL"`
explicitly to every command and skip the flag entirely when the var is
unset.

## S3 layout (per pipeline)

```
s3://$S3_BUCKET/pipelines/<slug>/
├── pipeline.json
├── raw/<container-prefix>/...      # CSVs you upload
├── clean/<table>/year=YYYY/month=MM/data.parquet   # worker output, read-only
├── jobs/<job-id>.json
└── dashboards/<id>.json
```

## List existing pipeline slugs

```sh
aws s3 ls "s3://$S3_BUCKET/pipelines/"
# CommonPrefixes -- one entry per pipeline slug.
```

To check whether a specific slug is taken:

```sh
aws s3 ls "s3://$S3_BUCKET/pipelines/my-pipeline/pipeline.json"
# Exits 0 with a line if present, 1 with no output if absent.
```

## Read the current config

```sh
aws s3 cp "s3://$S3_BUCKET/pipelines/my-pipeline/pipeline.json" - \
  | jq .
# `-` streams to stdout. Pipe through `jq` for sanity.
```

## Write the config

```sh
aws s3 cp pipeline.json \
  "s3://$S3_BUCKET/pipelines/my-pipeline/pipeline.json" \
  --content-type application/json
```

The PUT is unconditional. There's no S3-side ETag check the skill can
rely on (RustFS is inconsistent about honoring `If-Match`), so confirm
with the user before clobbering an existing slug.

## Trigger a run

Don't call the worker directly. The bucket webhook
(`pipelines/<slug>/raw/...` ObjectCreated → `/api/events/s3` →
debounced job) handles this:

```sh
# This both lands the data and triggers a run. The webhook waits
# briefly for additional uploads before firing.
aws s3 cp transactions-2026-01.csv \
  "s3://$S3_BUCKET/pipelines/my-pipeline/raw/transactions/"
```

If the user wants to re-run without uploading anything, they have to
click "Run" on the jobs page in the browser. There's no scriptable
re-run path.

## Watch a run finish

```sh
aws s3 ls "s3://$S3_BUCKET/pipelines/my-pipeline/jobs/" \
  | sort -r | head -n 5
# Jobs are timestamped via the key. Newest first after sort -r.

aws s3 cp \
  "s3://$S3_BUCKET/pipelines/my-pipeline/jobs/<id>.json" - \
  | jq '{status, startedAt, finishedAt, errors}'
```

A job is `completed` or `failed`; `errors[]` is empty on success.

## Delete a pipeline

```sh
aws s3 rm "s3://$S3_BUCKET/pipelines/my-pipeline/" --recursive
```

Removes the config, raw inputs, clean outputs, jobs, and dashboards.
Irreversible -- ask the user before doing this.
