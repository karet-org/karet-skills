---
name: karet-mapping-author
description: |
  Given a source container and a target Analytic Table, propose
  AST-JSON mapping expressions that translate source rows into target
  schema rows. Picks parse_date / cast / upper / lookup_ref based on
  the source/target type pair.
---

# karet-mapping-author

## When to invoke

The user has a source and a target table and needs the `mappings[]`
entry that connects them. Often called from inside
`karet-pipeline-builder`, but also useful standalone when a user adds
a new column to an existing Analytic Table and needs to update the
mapping.

## What you do

1. **Read both schemas.** The source container's columns (whatever
   types the user declared, usually `string`) and the target table's
   columns (canonical types).
2. **For each target column**, derive the simplest expression:
   - same name, same string-ish type → `{ kind: "col", name }`
   - same name, target=`date`, source=`string` → wrap in `parse_date`,
     ask the user for the format if unsure
   - same name, target=`int64|float64`, source=`string` → wrap in
     `cast`. If sample values have whitespace or commas, also wrap in
     `trim` and warn about formatting
   - target=`string`, source has mixed case → wrap in `upper` (most
     reliable for downstream lookup matching)
   - target column doesn't appear in source → ask the user what
     should fill it. Common cases: a `lookup_ref` against a known
     LookupMapping, a derived `concat`, or a literal default.
3. **For lookup-derived columns**, ask:
   - which LookupMapping to use,
   - what column to feed in,
   - whether to upper-case before lookup (almost always yes).
4. **Pick a partition column.** Default to the first `date`-typed
   target column with `granularity: "month"`. Confirm.
5. **Emit the mapping.** See `references/mapping-shape.md`.

## Hard rules

- **MUST** preserve the order of `Mapping.columns` to match the
  target table's `schema` order.
- **MUST NOT** invent target columns. If the user has 5 in the target
  and you map 4, leave the fifth blank with a `null` and call it out.
- **MUST** ask before guessing a date format. `%Y-%m-%d` and
  `%m/%d/%Y` look identical for `04/05/2026`.

## Output

```
Generated mapping `transactions_mapping`:
  source: transactions_raw
  target: transactions
  partition: date (month)
  6 columns mapped, 1 column needs your input (`category` -- which lookup?)
  3 warnings:
    - `amount` source has thousands separators; added trim
    - `date` format guessed as %Y-%m-%d on 100% of sampled rows
    - `description` upper-cased before lookup_ref
```
