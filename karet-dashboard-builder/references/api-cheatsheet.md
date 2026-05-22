# How the dashboard skill writes data

Dashboards live as plain JSON in S3 at
`pipelines/<slug>/dashboards/<dashboard-id>.json`. Karet's web service
reads them on demand -- there's no PUT route to go through. The skill
writes the file directly to the S3-compatible store (rustfs in dev,
real S3 in prod) and the next time the user visits
`/p/<slug>/dashboards/<dashboard-id>` it appears.

## Required env vars

The same ones the running web container uses -- find them in `.env`:

- `S3_BUCKET` (default `karet-data`)
- `S3_ENDPOINT` (default `http://localhost:9000` for local dev; absent
  in prod -- the SDK hits real S3)
- `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`

## Read the pipeline config (over HTTP, to inspect the table schema)

The web API stays the right path for *reads* -- it gives you parsed,
typed responses with no S3-credentials handling.

```sh
curl -sS "$KARET_BASE_URL/api/p/<slug>/config" \
  -H "x-api-key: $KARET_API_KEY"
# -> full PipelineConfig. Look at `analytic_tables[].schema` to see
#    what columns the dashboard can reference.
```

## List existing dashboards (over HTTP)

```sh
curl -sS "$KARET_BASE_URL/api/p/<slug>/dashboards" \
  -H "x-api-key: $KARET_API_KEY"
# -> {"dashboards":["spending_overview","monthly_summary"]}
```

## Sample real data (over HTTP, recommended before authoring)

```sh
curl -sS "$KARET_BASE_URL/api/p/<slug>/tables/<table>/rows?limit=20" \
  -H "x-api-key: $KARET_API_KEY"
```

## Write the dashboard (directly to S3)

```sh
aws s3 cp dashboard.json \
  "s3://$S3_BUCKET/pipelines/<slug>/dashboards/<dashboard-id>.json" \
  --endpoint-url "$S3_ENDPOINT" \
  --content-type application/json
```

Or with the AWS SDK in any language. The S3 key format is the
contract; nothing else cares.

A few rules to keep the file consistent with how the renderer reads it:

- The S3 key is `pipelines/<slug>/dashboards/<dashboard-id>.json` --
  no nesting, no trailing whitespace in the id.
- `<dashboard-id>` should equal the JSON's `id` field; the renderer
  keys off the URL slug, but other tooling (the dashboards menu, the
  template export path) reads `id` from the body.
- Use `Content-Type: application/json`.

## Delete a dashboard

```sh
aws s3 rm \
  "s3://$S3_BUCKET/pipelines/<slug>/dashboards/<dashboard-id>.json" \
  --endpoint-url "$S3_ENDPOINT"
```
