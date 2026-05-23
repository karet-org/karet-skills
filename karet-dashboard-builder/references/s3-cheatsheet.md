# Karet dashboard S3 cheatsheet

Dashboards live as plain JSON in S3 at
`pipelines/<slug>/dashboards/<dashboard-id>.json`. Karet's web service
reads them on each page load -- there's no PUT route to go through.
Both reading and writing go directly to the S3-compatible store
(RustFS in dev, real S3 in prod).

## Required environment

The same vars Karet itself reads -- find them in `.env`:

| Variable | Notes |
|----------|-------|
| `S3_BUCKET` | Bucket name. Default `karet-data`. |
| `AWS_ENDPOINT_URL` | S3 endpoint (e.g. `http://localhost:9000` for local RustFS). **Unset** for real AWS S3. |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | S3 credentials. |
| `AWS_REGION` | Required even for RustFS. Use `us-east-1` if you don't care. |

The AWS CLI honors `AWS_ENDPOINT_URL` automatically (since v2.13). On
older versions, pass `--endpoint-url "$AWS_ENDPOINT_URL"` explicitly
when it's set, and skip the flag entirely when it isn't.

## Read the pipeline config (to inspect the table schema)

```sh
aws s3 cp \
  "s3://$S3_BUCKET/pipelines/<slug>/pipeline.json" - \
  | jq '.analytic_tables[] | select(.id == "<table>") | .schema'
```

## List existing dashboards

```sh
aws s3 ls "s3://$S3_BUCKET/pipelines/<slug>/dashboards/"
```

Each `*.json` listed is one dashboard. The filename stem is the
dashboard id and matches the URL slug at
`<site>/p/<slug>/dashboards/<id>`.

## Sample real data (parquet)

The worker writes partitioned Parquet under
`pipelines/<slug>/clean/<table>/year=YYYY/month=MM/data.parquet`.
Pull one file down and inspect it locally:

```sh
aws s3 cp \
  "s3://$S3_BUCKET/pipelines/<slug>/clean/<table>/year=2026/month=04/data.parquet" \
  /tmp/sample.parquet

duckdb -c "SELECT * FROM '/tmp/sample.parquet' LIMIT 20"
# Or: parquet-tools head /tmp/sample.parquet
# Or: python -c "import polars as pl; print(pl.read_parquet('/tmp/sample.parquet').head(20))"
```

If `clean/` is empty, the pipeline hasn't run successfully yet -- skip
the sample step and proceed without empirical data.

## Read an existing dashboard

```sh
aws s3 cp \
  "s3://$S3_BUCKET/pipelines/<slug>/dashboards/<id>.json" - \
  | jq .
```

## Write the dashboard

```sh
aws s3 cp dashboard.json \
  "s3://$S3_BUCKET/pipelines/<slug>/dashboards/<id>.json" \
  --content-type application/json
```

A few rules to keep the file consistent with how the renderer reads it:

- The S3 key is `pipelines/<slug>/dashboards/<dashboard-id>.json` --
  no nesting, no trailing whitespace in the id.
- `<dashboard-id>` should equal the JSON's `id` field. The renderer
  keys off the URL slug, but other tooling (the dashboards menu, the
  template export path) reads `id` from the body.
- Always send `--content-type application/json`.

## Delete a dashboard

```sh
aws s3 rm \
  "s3://$S3_BUCKET/pipelines/<slug>/dashboards/<id>.json"
```
