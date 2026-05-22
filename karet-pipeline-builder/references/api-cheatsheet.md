# Karet web API cheatsheet (skill-relevant subset)

Base URL: `$KARET_BASE_URL` (default `http://localhost:3000`).
Auth: `x-api-key: $KARET_API_KEY` on every request.

## Create a pipeline

```sh
curl -sS -X POST "$KARET_BASE_URL/api/pipelines" \
  -H "x-api-key: $KARET_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"slug":"my-pipeline","template":"blank"}'
# -> {"ok":true,"pipeline":"my-pipeline"}
# -> 409 {"error":"already_exists"} if the slug is taken
```

`template: "blank"` writes an empty `pipeline.json` so we can PUT the
real one next. `template: "spending"` writes the seed Spending Tracker
shape -- not what this skill wants.

## Read the current config (and ETag)

```sh
curl -sS -i "$KARET_BASE_URL/api/p/my-pipeline/config" \
  -H "x-api-key: $KARET_API_KEY"
# Pull the `ETag:` response header. Strip surrounding quotes.
```

## Write the config

```sh
curl -sS -X PUT "$KARET_BASE_URL/api/p/my-pipeline/config" \
  -H "x-api-key: $KARET_API_KEY" \
  -H "Content-Type: application/json" \
  -H 'If-Match: "<etag-from-GET>"' \
  --data-binary @pipeline.json
# -> {"ok":true,"etag":"..."}
# -> 412 if the ETag doesn't match (someone else wrote between your GET and PUT)
```

## Trigger a run

```sh
curl -sS -X POST "$KARET_BASE_URL/api/p/my-pipeline/jobs" \
  -H "x-api-key: $KARET_API_KEY"
# -> {"job":{"id":"...","status":"running",...}}
```

Add `?clean=true` to wipe the analytic_tables' output prefix before
running.

## Watch a run finish

Poll `GET /api/p/<slug>/jobs` every 2 seconds until the newest job is
`completed` or `failed`. Job records sort newest-first by `startedAt`.

## Validate before you write (optional)

If you want to catch shape errors before persisting:

```sh
curl -sS -X POST "$KARET_BASE_URL/api/p/my-pipeline/validate" \
  -H "x-api-key: $KARET_API_KEY" \
  -H "Content-Type: application/json" \
  --data-binary @pipeline.json
# -> {"ok":true} | {"ok":false,"errors":[{"message":"..."}]}
```

This calls the worker's validation endpoint; same logic that runs
before save in the graph editor.
