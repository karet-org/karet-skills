---
name: karet-dashboard-tweaker
description: |
  Edit an existing Karet dashboard in place. Pulls the JSON from S3,
  applies the user's requested change ("add a top-3 doughnut", "switch
  to monthly bins", "rename Total to Total Spent"), and writes it back
  to S3.
---

# karet-dashboard-tweaker

## When to invoke

The user has a dashboard and wants to modify it without opening the
JSON themselves. Examples:

- "Add a doughnut by `account` to my spending dashboard"
- "Switch the line chart to monthly bins"
- "Use CAD for all the KPIs"
- "Drop the Top Merchants panel"
- "Make the table show 20 rows per page"

## What you do

1. **Read the S3 environment.** `S3_BUCKET`, `AWS_ENDPOINT_URL` (only
   for RustFS / MinIO), `AWS_REGION`, plus the access keys. See
   `karet-dashboard-builder/references/s3-cheatsheet.md`.
2. **Identify the target.** Pipeline slug + dashboard id. If unclear,
   ask. List candidates with `aws s3 ls
   s3://$S3_BUCKET/pipelines/<slug>/dashboards/`.
3. **Read the current dashboard.**

   ```sh
   aws s3 cp \
     "s3://$S3_BUCKET/pipelines/<slug>/dashboards/<id>.json" \
     /tmp/dashboard.json
   ```

4. **Read the table schema.**

   ```sh
   aws s3 cp \
     "s3://$S3_BUCKET/pipelines/<slug>/pipeline.json" - \
     | jq '.analytic_tables[] | select(.id == "<table>") | .schema'
   ```

   `<table>` is the dashboard's `analytic_table_id`. Required for any
   change that references a column.
5. **Plan the edit.** Common edits:
   - **Add panel** -- append to `panels[]`. Use defaults from
     `karet-dashboard-builder/references/panel-recipes.md`.
   - **Remove panel** -- match by `title` or `kind`+key column.
     Confirm with the user if multiple panels match.
   - **Change aggregation / bin / format** -- mutate the matching
     panel in place.
   - **Rename** -- only the `title` field. Don't change the panel
     `kind`.
   - **Reorder** -- swap entries in `panels[]`. Mention that grid
     placement (`gridColumn` etc.) might need adjusting.
6. **Validate.** Every column referenced by every panel must exist in
   the table schema. Refuse to save if not.
7. **Write back.**

   ```sh
   aws s3 cp /tmp/dashboard.json \
     "s3://$S3_BUCKET/pipelines/<slug>/dashboards/<id>.json" \
     --content-type application/json
   ```

   The PUT is unconditional -- there's no S3-side ETag check the skill
   can rely on. Only edit the fields you intended to touch and write
   the rest verbatim.
8. **Confirm.** Print the diff of what changed in human-readable form.
   Ask for the user's site URL/domain if you don't know it yet, then
   tell them to reload `<site>/p/<slug>/dashboards/<id>`.

## Hard rules

- **MUST NOT** rewrite panels you didn't intend to change. Read,
  edit one or more entries, write the rest verbatim.
- **MUST NOT** invent column names. Validate every reference against
  the live `pipeline.json` schema.
- **MUST** preserve the dashboard's `id` field. The URL slug, the S3
  key, and the `id` need to stay in sync.
- **MUST NOT** assume there's a Karet HTTP API for dashboards. Reads
  and writes both go through S3.
- **MUST NOT** read or write `_auth/admin.json` or anything under
  `clean/`.

## Output

```
Edited `monthly_overview`:
  + Added doughnut "By Account" (group_by=account, value=amount, sum)
  ~ Changed line "Monthly Trend" bin: month -> day
  - Removed bar "Top Merchants"

Wrote s3://$S3_BUCKET/pipelines/monthly-spending/dashboards/monthly_overview.json.
Reload <site>/p/monthly-spending/dashboards/monthly_overview to see the change.
```
