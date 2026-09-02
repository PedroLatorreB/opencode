---
name: vault-notes
description: Use when the user says "vault", "notas", "Obsidian", "actualiza nota", "guarda en stas", "documenta requerimiento", "marca estado QA", or asks to sync notes with Engram. Creates and updates Markdown notes in /home/platorre/vaults/stas/ and records summaries in Engram.
---

# Vault Notes

Use this skill to create or update the user's Markdown notes vault at `/home/platorre/vaults/stas/`. The vault is used from Neovim and Obsidian-compatible tooling, not the Obsidian desktop app, so keep files plain Markdown with YAML frontmatter and portable links.

## Core Rules

- Vault root: `/home/platorre/vaults/stas/`.
- Prefer existing notes before creating new ones. Search by project, change name, tags, status, and likely filename.
- Create new notes from files in `/home/platorre/vaults/stas/templates/` when an appropriate note does not exist.
- Maintain frontmatter fields: `project`, `type`, `change`, `status`, `tags`, `created`, and `updated`.
- Use status values from this vocabulary: `idea`, `exploracion`, `propuesto`, `implementando`, `qa`, `en_revision`, `bloqueado`, `finalizado`, `archivado`.
- Append or update a dated `## Log` section instead of overwriting user-authored content.
- Keep pending tasks and current status visible in the note body.
- Use Engram for concise machine memory and the Vault for human-readable notes.
- Keep Vault and Engram synchronized when meaningful: save concise Engram observations for decisions, bug fixes, discoveries, conventions, and completed work; include Engram IDs in notes when available.
- Never claim final closure unless the user explicitly confirms the work is finalizado or archivado.
- Do not configure MCP unless the user explicitly asks for it.
- For vault file creation and updates, prefer OpenCode `write` and `edit` tools with absolute paths under `/home/platorre/vaults/stas/`.
- Do not use `bash` redirection, heredocs, `tee`, `cp`, `mv`, or `rm` to create or mutate vault notes unless the user explicitly asks for shell-based file operations.
- If OpenCode asks for external directory approval, request/accept only the narrow path-scoped permission for `~/vaults/stas/**`; do not broaden to the whole home directory.
- If an external directory request is rejected, stop retrying the same tool call and return a proposed patch or exact manual edit instead.

## Folder Conventions

- `notes/`: PRDs, convention notes, general project notes, and README-style documentation.
- `qa/`: QA checklists and verification notes.
- `decisions/`: decision records and tradeoffs.
- `bugs/`: bug reports, investigations, fixes, and regressions.
- `daily/`: dated work logs.
- `templates/`: reusable Markdown templates.

## Common Triggers

- "vault"
- "notas"
- "Obsidian"
- "actualiza nota"
- "guarda en stas"
- "documenta requerimiento"
- "marca estado QA"
- "sincroniza con Engram"
- "crea PRD"
- "registra decision"
- "anota bug"

## Workflow

1. Identify the target project, change, note type, and desired status.
2. Search the vault for existing notes using filenames, frontmatter terms, tags, and headings.
3. If no suitable note exists, create one from the matching template.
4. Update frontmatter conservatively, preserving user fields and content.
5. Append a dated log entry under `## Log` with the change made, evidence, and pending tasks.
6. Save or update Engram only for concise durable memory, then add Engram IDs to the note if available.
7. Report the files changed and whether the note remains pending, in QA, blocked, or ready for user confirmation.

## Requirement Considerations Standard

For every requirement, PRD, feature, QA-to-PROD, or deployment-oriented note, maintain a parent `## Consideraciones` section with these subsections:

### WEB
- Branch or source change to merge/build.
- Environment/build consideration.
- Local or test-only files that must be excluded.
- Minimal WEB validation.

### API
- Branch or API change to merge/deploy.
- Shared libraries/DLLs/contracts that must travel together.
- Endpoint or compatibility validation.
- Use `No aplica.` if the requirement has no API impact.

### DB
- Scripts/SP/tables involved.
- Mark database artifacts with `(nuevo)`, `(modificado)`, or `(ALTER)` when applicable.
- Execution order or dependency summary.
- Use `No aplica.` if the requirement has no DB impact.

### GIT
- Source branch and destination branch.
- Files to exclude from commit/merge.
- Short pre-merge checklist.

Keep these bullets concise and operational. Prefer overview over implementation details unless the user explicitly asks for a deep release checklist.

## Note-Type Guidance

### PRD Notes

- Store in `notes/` unless the user requests another folder.
- Use `templates/prd.md` for new files.
- Capture summary, context, goals, non-goals, requirements, acceptance criteria, open questions, status, and related Engram IDs.
- Keep requirements and acceptance criteria as checklists when possible.
- Always include/update `## Consideraciones` with `WEB`, `API`, `DB`, and `GIT` subsections using the Requirement Considerations Standard.

### QA Notes

- Store in `qa/`.
- Use `templates/qa-checklist.md` for new files.
- Mark status as `qa`, `en_revision`, `bloqueado`, or `finalizado` only when evidence supports it.
- Include commands run, files reviewed, findings, regressions, and unresolved risks.

### Decision Notes

- Store in `decisions/`.
- Use `templates/decision.md` for new files.
- Capture decision, context, options considered, rationale, consequences, and related Engram observation IDs.
- Save a matching Engram decision or architecture observation when the decision is durable.

### Bug Notes

- Store in `bugs/`.
- Use `templates/bug.md` for new files.
- Capture reproduction, expected behavior, actual behavior, root cause, fix, verification, status, and related Engram IDs.
- Save a matching Engram bugfix observation after the fix is completed.

### Daily Notes

- Store in `daily/` with a date-first filename such as `YYYY-MM-DD.md` or `YYYY-MM-DD-project.md`.
- Use `templates/daily.md` for new files.
- Capture focus, work log, decisions, QA, blockers, pending tasks, and related Engram session IDs.

## Filename Guidance

- Use lowercase kebab-case for descriptive notes.
- Prefer filenames like `project-change-prd.md`, `project-change-qa.md`, `project-change-decision.md`, and `project-bug-short-title.md`.
- Daily notes should start with `YYYY-MM-DD`.

## Safety

- Do not delete or move vault files unless the user explicitly asks.
- Do not overwrite user-written sections; append logs or make targeted edits.
- Do not rely on Obsidian desktop-only features.
- Do not expose private note content beyond what is needed to answer the user.
