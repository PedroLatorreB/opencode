---
name: sql-review
description: Use when working with SQL, SELECT *, COUNT(*), queries, migrations, repositories, ORMs, indexes, joins, database performance, or visible SQL bad practices. Reviews risky SQL patterns and proposes pragmatic fixes.
---

# SQL Review

Use this skill whenever the task involves SQL directly or indirectly: handwritten queries, migrations, database repositories, query builders, ORM-generated SQL, reporting queries, stored procedures, views, indexes, or database performance.

## Core Rule

Never let `SELECT *` or `COUNT(*)` pass silently.

When either appears, call it out explicitly, explain the risk, and propose a concrete alternative. Do not rewrite SQL blindly; correctness and intent come first.

## Patterns to Flag

Actively look for these visible SQL issues:

- `SELECT *`
- `COUNT(*)` used without clear intent
- existence checks implemented as counts
- unbounded reads where result size may grow
- `UPDATE` or `DELETE` without a restrictive `WHERE`
- implicit joins or unclear join conditions
- N+1 query patterns
- duplicated subqueries
- non-sargable predicates, such as wrapping indexed columns in functions
- filtering, joining, or ordering on columns that likely need indexes
- broad `LIKE '%term%'` searches on large tables
- pagination without deterministic ordering
- `ORDER BY` on expensive expressions
- unnecessary `DISTINCT` used to hide join mistakes
- ORM/query-builder code that likely generates inefficient SQL

## How to Review

For each issue found:

1. Name the issue.
2. Explain why it matters.
3. Ask whether the behavior is intentional only when intent is unclear.
4. Propose a safer or more explicit alternative.
5. Mention tradeoffs when the alternative is not universally better.

## Guidance for `SELECT *`

Treat `SELECT *` as a potential maintainability and performance problem.

Prefer explicit column lists because they:

- avoid over-fetching data
- reduce network and memory usage
- make API/data contracts clearer
- prevent accidental exposure of newly added columns
- reduce breakage when schemas evolve

Accept `SELECT *` only when there is a clear reason, such as temporary debugging, throwaway exploration, or intentionally selecting all columns in a controlled internal script.

## Guidance for `COUNT(*)`

Do not assume `COUNT(*)` is always wrong, but always verify intent.

Common alternatives:

- Use `EXISTS` for existence checks.
- Use `LIMIT 1` patterns when supported and appropriate.
- Count a specific indexed column only when it matches the intended semantics.
- Use cached counters for hot paths.
- Use approximate counts for analytics or dashboards when exactness is not required.
- Add or validate indexes for count filters on large tables.

Be careful: in many database engines, `COUNT(*)` is semantically correct and can be optimized. The point is not to ban it blindly; the point is to force an explicit decision.

## Response Style

Be direct and helpful:

- Start with the concrete finding.
- Keep the explanation practical.
- Offer a fix the team can actually apply.
- If there are multiple options, list the tradeoffs briefly.

Example:

```sql
SELECT * FROM patients WHERE active = true;
```

Review:

```text
Issue: SELECT * over-fetches columns and makes the result contract implicit.
Suggestion: select only the fields needed by the caller, for example id, name, status.
Tradeoff: if this is a one-off debugging query, SELECT * may be acceptable, but not in production code.
```
