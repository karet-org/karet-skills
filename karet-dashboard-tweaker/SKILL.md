---
name: karet-dashboard-tweaker
description: |
  Edit an existing Karet dashboard in place. Reads the current JSON,
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

1. **Identify the target.** Pipeline slug + dashboard id. If unclear,
   ask. List dashboards via the web API if helpful.
2. **Read the current dashboard.** `GET /api/p/<slug>/dashboards/<id>`.
3. **Read the table schema.** `GET /api/p/<slug>/config` and look at
   `analytic_tables[].schema` for the dashboard's
   `analytic_table_id`. Required for any change that references a
   column.
4. **Plan the edit.** Common edits:
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
5. **Validate.** Every column referenced by every panel must exist in
   the table schema. Refuse to save if not.
6. **Write back.** Save the edited JSON directly to S3 at
   `pipelines/<slug>/dashboards/<id>.json`. Karet has no PUT route
   for dashboards -- they're plain S3 objects. See
   `karet-dashboard-builder/references/api-cheatsheet.md` for env vars
   and the `aws s3 cp` recipe.
7. **Confirm.** Print the diff of what changed in human-readable form
   and the URL to refresh.

## Hard rules

- **MUST NOT** rewrite panels you didn't intend to change. Read,
  edit one or more entries, write the rest verbatim.
- **MUST NOT** invent column names. Validate every reference against
  the live schema.
- **MUST** preserve the dashboard's `id` field. The URL slug and
  the `id` need to stay in sync.

## Output

```
Edited `monthly_overview`:
  + Added doughnut "By Account" (group_by=account, value=amount, sum)
  ~ Changed line "Monthly Trend" bin: month -> day
  - Removed bar "Top Merchants"

Reload http://localhost:3000/p/monthly-spending/dashboards/monthly_overview
```
