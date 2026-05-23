# karet-skills

Claude skills that automate common tasks against a self-hosted
[Karet](https://github.com/karet-org/karet) instance. Each skill is a
self-contained directory the user can drop into `~/.claude/skills/`
(or symlink in place during development) and invoke from a Claude
Code session.

## Audience

These skills are for people **self-hosting** Karet. Karet has no
public HTTP API for managing pipelines or dashboards -- the web app
reads its config out of S3 on every page load, and the worker reads
the same config when it runs. So every skill here treats S3 as the
source of truth and writes config directly with the AWS CLI.

If you need to *diagnose* something on the running site (a panel that
looks empty, a dashboard 500ing, etc.), the skill will ask for the
URL or domain you reach the site at -- only to point you at the right
page -- and use S3 for the actual data inspection.

## Configuration

The skills shell out to `aws s3 ...`, so they need the same AWS
credentials and endpoint settings the running Karet stack uses. Pull
these out of the `.env` you start the stack with:

| Variable | Used for |
|----------|----------|
| `S3_BUCKET` | The bucket Karet writes into. Default `karet-data`. |
| `AWS_ENDPOINT_URL` | S3 endpoint. Set to `http://localhost:9000` (or wherever your RustFS / MinIO sits) for local dev; **leave unset** when pointing at real AWS S3. |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | S3 credentials. |
| `AWS_REGION` | S3 region. Required even for RustFS -- use `us-east-1` if you don't care. |

Skills also occasionally need to know the public URL of your Karet
site (to print a "open this page to verify" link). They'll ask for
it the first time they need it; there's no environment variable for
this.

## S3 layout that the skills read and write

```
s3://$S3_BUCKET/
├── _auth/admin.json               # admin password -- DO NOT TOUCH
└── pipelines/<slug>/
    ├── pipeline.json              # PipelineConfig (the thing you edit)
    ├── raw/<container-prefix>/    # CSVs you upload; webhook triggers a run
    ├── clean/<table>/             # Parquet output from the worker
    │   └── year=YYYY/month=MM/data.parquet
    ├── jobs/<job-id>.json         # Run records
    └── dashboards/<id>.json       # DashboardConfig
```

Two things the skills will never touch:

- `_auth/admin.json` -- the scrypt-hashed admin password. Do not read
  or write it; rotate via the site's settings page.
- `clean/` -- worker output. Treat it as read-only. The worker
  rewrites it on every run.

## Quick orientation

- **Building a new pipeline from CSV(s)**: `karet-pipeline-builder`
  (or `karet-pipeline-importer` for a whole folder).
- **Writing a dashboard for a populated table**: `karet-dashboard-builder`.
- **Tweaking an existing dashboard**: `karet-dashboard-tweaker`.
- **Figuring out why a panel is empty**: `karet-data-detective`.
- **Profiling a CSV before you touch a pipeline**: `karet-csv-inspector`.
- **Authoring lookups, mappings in isolation**: `karet-lookup-builder`,
  `karet-mapping-author`.
