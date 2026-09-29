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
- `SELECT 1` in `EXISTS` or subqueries where the primary key expresses intent better
- `COUNT(*)` used without clear intent
- existence checks implemented as counts
- commented-out code without context (dead code)
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
- temporary tables without guarded cleanup (`DROP TABLE #tmp`)
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

Accept `SELECT *` only when there is a clear reason, such as temporary debugging, throwaway exploration, or intentionally selecting all columns in a controlled internal script. Throwaway scope ends there: any production object (SP, view, trigger, function, deployed migration, app/repository query) falls under veto A1 with no throwaway exception.

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

## Guidance for `SELECT 1`

Flag `SELECT 1` inside `EXISTS` or subqueries and recommend `SELECT [PRIMARY_KEY]` instead.

Prefer the primary key (e.g. `EXISTS (SELECT id ...)` or the corresponding PK) because it:

- makes intent explicit about which entity must exist
- avoids ambiguity about what is being checked
- keeps reviews consistent with the explicit-column rule used for `SELECT *`

Example:

```sql
-- Before
SELECT * FROM patients p WHERE EXISTS (SELECT 1 FROM admissions a WHERE a.patient_id = p.id);

-- After
SELECT p.id, p.name FROM patients p WHERE EXISTS (SELECT a.id FROM admissions a WHERE a.patient_id = p.id);
```

Accept `SELECT 1` only in throwaway scripts where brevity matters more than clarity. Idiomatic fix is `SELECT <PK>` of the checked table (e.g. `SELECT a.id`); do not flag `SELECT <PK>` inside `EXISTS` as a violation.

## Guidance for Commented SQL

Distinguish dead code from valuable comments. Flag for removal only when all apply:

- contains executable SQL, old queries, or query fragments
- has no explanation of why it is kept
- has no TODO/FIXME, ticket reference, date, or condition for reactivation

Leave intact comments that add value:

- explain complex logic, non-obvious business rules, or workarounds
- document intent, edge cases, or performance rationale
- carry actionable TODO/FIXME with context (what is pending and why)
- reference tickets, dates, or decisions needed to understand the code

When flagging dead code, propose deletion, not preservation. When in doubt about intent, ask once before recommending removal.

## Guidance for Temporary Tables

Flag any temporary table (`#tablaTemporal`) that lacks guarded cleanup. Require this pattern before (re)creating each temp table:

```sql
IF OBJECT_ID('TEMPDB.DBO.#tablaTemporal', 'U') IS NOT NULL
  DROP TABLE #tablaTemporal;
```

Require every `#temp` table created in a Stored Procedure to be dropped inside the same `BEGIN/END` block of the SP. Flag temp tables with no matching `DROP TABLE` inside `BEGIN/END`, or dropped only outside it.

## SQL Server Stored Procedures

When the user asks to generate, create, or modify a SQL Server Stored Procedure/SP, use this template as the base structure. Parametrize the database: replace `[DB]` with the target database (e.g. `CLINIWIN`); never assume `CLINIWIN` when the user names another DB or works multi-clínica.

```sql
USE [DB]
GO

SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

-- =============================================
-- NOMBRE DEL OBJETO  : spu_
-- TIPO DE OBJETO     : Procedimiento Almacenado
-- FECHA CREACIÓN     : 04/09/2026
-- AUTOR              : Pedro Latorre <pedro.latorre@andessalud.cl>
-- DESCRIPCION        :
--
-- PARAMETROS         :
--
-- MODIFICADO POR     :
-- FECHA              :
-- OBSERVACIÓN        :
--
-- =============================================
--exec spu_
CREATE OR ALTER PROCEDURE [dbo].[spu_]

AS
BEGIN
    SET XACT_ABORT ON

    BEGIN TRY
        -- BEGIN TRAN only when the SP writes (INSERT/UPDATE/DELETE).
        -- COMMIT inside TRY; ROLLBACK guarded by IF @@TRANCOUNT > 0 inside CATCH.
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK;
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrorSeverity INT = ERROR_SEVERITY();
        DECLARE @ErrorState INT = ERROR_STATE();
        RAISERROR(@ErrorMessage, @ErrorSeverity, @ErrorState);
        THROW;
    END CATCH
END
GO

GRANT VIEW DEFINITION ON spu_ TO ROL_STAS_VIEW
GO
GRANT EXECUTE ON spu_ TO PUBLIC
GO
```

Stored Procedure generation rules:

- Replace every `spu_` placeholder with the real procedure name, including the header, `CREATE OR ALTER PROCEDURE`, execution example, and `GRANT` statements.
- Keep `USE [DB]`, `SET ANSI_NULLS ON`, and `SET QUOTED_IDENTIFIER ON` unless the user explicitly requests another database or session setting. `USE [DB]` must name the target database explicitly; default to `CLINIWIN` only when the user gives no other DB.
- Complete `DESCRIPCION` and `PARAMETROS` when the procedure intent and parameters are known.
- Preserve `CREATE OR ALTER PROCEDURE` for idempotent deployment scripts.
- Preserve `SET XACT_ABORT ON`.
- If explicit transactions are added, handle `BEGIN TRAN`, `COMMIT`, and `ROLLBACK` correctly inside `TRY/CATCH` (see block D: `COMMIT` in `TRY`, `IF @@TRANCOUNT > 0 ROLLBACK` + capture `ERROR_MESSAGE()` / `ERROR_SEVERITY()` / `ERROR_STATE()` + `RAISERROR` + `THROW` in `CATCH`).
- If the Stored Procedure uses temporary tables, guard each one with `IF OBJECT_ID('TEMPDB.DBO.#tablaTemporal', 'U') IS NOT NULL DROP TABLE #tablaTemporal;` before creation and drop every `#temp` table inside the `BEGIN/END` block of the SP.
- If the Stored Procedure contains queries, apply the normal SQL review rules plus STAS blocks A-F below: veto `SELECT *`, veto `COUNT(*)` (only `COUNT(campo)` unless block A documents an exception), check for non-sargable predicates, missing indexes, unbounded reads, and broad updates/deletes.
- `USE [DB]` is the single placeholder for the target database in all blocks (A-F, D2, F1); never use `USE [BASE_DATOS]`. Multi-clínica: repeat the script per clinic DB with its explicit name; each repetition keeps its own `USE [DB]` + `GO`.
- Table/audit example for blocks B4/B5 (adapt names per case):

```sql
EXEC sp_addextendedproperty
  @name = N'MS_Description', @value = N'Identificador unico del paciente',
  @level0type = N'SCHEMA', @level0name = N'dbo',
  @level1type = N'TABLE', @level1name = N'PACIENTES',
  @level2type = N'COLUMN', @level2name = N'PAC_ID';
GO

-- Audit table mirrors origin fields + AUD_TIPO (INS/UPD/DEL via trigger tr_<mod>_<tabla>_INS/UPD/DEL).
```

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

## STAS-A — Vetos binarios

Binary verdicts. No silent pass. Each veto is fail-closed unless its documented exception applies.

- A1 `SELECT *` / `SELECT <alias>.*`: veto in production objects. Require explicit column list. See existing `Guidance for SELECT *`; the throwaway allowance there applies only to ad-hoc debugging/exploration scripts, never to production objects.
- A2 `COUNT(*)`: veto. Require `COUNT(campo)`; see STAS-A exception below (T2).
- A3 `LIKE '%x%'` / leading-`%` on large tables: veto except bounded searchers per block C.
- A4 Global temp `##`: veto. Require local `#` with guarded `DROP` or table variable; see existing `Guidance for Temporary Tables`.
- A5 Physical delete without logical-state column: veto. Require status field; no `DELETE` as erase.
- A6 Versioned object names (`spu_..._v2`, `_old`, `_bak`): veto unless justified with deletion of prior versions.
- A7 `UPDATE`/`DELETE` without restrictive `WHERE` or outside transaction (manual DML): veto; see block D.

## STAS-A excepción COUNT (T2)

`COUNT(*)` is allowed only when all hold, otherwise rewrite to `COUNT(campo)`:

1. Engine-optimized or semantic need is explicit (e.g. `COUNT(*)` over filtered PK with supporting index).
2. Intent documented inline (`-- COUNT(*) intencional: ...`) with ticket/date when available.
3. Existence checks still prefer `EXISTS` / `LIMIT 1`; hot-path counts prefer cached counters; analytics prefer approximate counts (see existing `Guidance for COUNT(*)`).

When in doubt, flag and ask once; do not silently approve legacy `COUNT(*)`.

Legacy migration: existing `COUNT(*)` in legacy code is flagged, not auto-rewritten. Migrate gradually — new code must use `COUNT(campo)` or a documented T2 exception; legacy `COUNT(*)` is rewritten only when its file is touched, with intent re-validated per T2.

## STAS-B — Nomenclatura y tablas

- B1 Compiled objects: `<abreviacion>_<modulo>_<descriptivo>` with `spu_` / `fn_` / `tr_` / `vi_`. Transversal module is `global`; abbreviate submodule to 3 letters. Audit triggers may omit descriptivo; action acronym in trigger name (`_INS` / `_UPD` / `_DEL`).
- B2 Tables: name in `MAYUSCULA` with `_` (e.g. `NOMBRE_COMPUESTO`). Columns: 3-letter prefix (initials if compound, first 3 if simple), self-descriptive with `_`, one field per line with type/length/nulls/default. Minimum one unique/autoincremental id key included in script.
- B3 `USE <DB>` + `GO` on table scripts; parametrize `[DB]`, never assume only `CLINIWIN` (multi-clínica: repeat per clinic, exception authorized and documented).
- B4 Mandatory `sp_addextendedproperty MS_Description` per column; adding a column requires default + extended property + DBA-agreed type.
- B5 Audit: requester states whether audited. Audit table mirrors origin fields + `AUD_TIPO`; audit triggers created with the table.

## STAS-C — Buscadores LIKE

Wildcards only inside searchers that are both documented and bounded:

- Documented: mark search-block start/end inline.
- Bounded: never wildcards-only; always a validated variable with minimum length 2 non-blank, non-wildcard chars (`LEN(@var) >= 2` after stripping blanks/`%`/`_`).
- Applies to regular, temp, and system tables. Otherwise flag `LIKE '%term%'` per existing patterns list.
- Performance doubt: test in non-production with DBA.

## STAS-D — Transaccionalidad y errores

- D1 Mandatory error handling on every `INSERT` / `UPDATE` / `DELETE`, via app or SQL. SP pattern is the template above: `SET XACT_ABORT ON`, `BEGIN TRY` + `COMMIT`, `BEGIN CATCH` + `IF @@TRANCOUNT > 0 ROLLBACK` + capture `ERROR_MESSAGE()` / `ERROR_SEVERITY()` / `ERROR_STATE()` + `THROW` / `RAISERROR`.
- D2 Manual DML (no app/maintainer): email + authorization, `USE [DB]` + `GO`, always in transaction(s), always `WHERE`, everything inside file(s); nothing outside files executes.
- D3 Cursors only with justification endorsed by JP and explicit authorization in the request.
- D4 `CREATE OR ALTER` preferred (or `DROP` in script) for reliable audit; SP modification appends `MODIFICADO POR / FECHA / OBSERVACIÓN` below last block, newest last; do not reassign permissions on modify.

## STAS-E — Higiene

- E1 Reserved words/instructions fully `MAYUSCULAS`.
- E2 No commented-out code to production. Allowed: functional comments explaining logic/rules/workarounds. Exception to keep commented code requires start/end mark + date/author/reason. See existing `Guidance for Commented SQL` for the dead-code test.
- E3 Prefer table variables for small sets (live only for the block or until `GO`); `#temp` keeps guarded `DROP` rule.
- E4 Close scripts with `GO` + `GRANT` (`GRANT VIEW DEFINITION ON <obj> TO ROL_STAS_VIEW`; add `GRANT EXECUTE ON <proc> TO PUBLIC` for SPs).
- E5 Table-affecting or locking changes run off-hours; Friday has no production passes.

## STAS-F — Checklist pre-paso

Verify each line before approving; any fail rejects the load:

| # | Check | Fail if |
|---|-------|---------|
| F1 | `USE [DB]` + `GO`, target DB explicit (multi-clínica covered) | missing / hardcoded wrong DB |
| F2 | `CREATE OR ALTER`, header + `MODIFICADO` chain newest-last | versioned name / missing header |
| F3 | Naming B1-B2, columns one-per-line + PK | wrong prefix / lowercase table |
| F4 | No veto A1-A7 open (or exception documented) | bare `*`, `COUNT(*)`, `##`, leading-`%` |
| F5 | Searchers bounded + documented (C) | wildcards-only / `LEN < 2` |
| F6 | `TRY/CATCH` + `@@TRANCOUNT` + `XACT_ABORT` on writes (D) | unwrapped DML / missing `WHERE` |
| F7 | `#` guarded + dropped in-block; else table variable (E3) | unguarded / leaked temp |
| F8 | Keywords upper, no dead comments (E1-E2) | lowercase verbs / commented SQL |
| F9 | `GO` + `GRANT VIEW DEFINITION TO ROL_STAS_VIEW` (+ `EXECUTE` for SP) | missing grant |
| F10 | Extended property + audit decision (B4-B5) | missing description / audit undefined |
