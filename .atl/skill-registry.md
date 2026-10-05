# Skill Registry

**Orchestrator use only.** Read this registry once per session to resolve skill paths, then pass pre-resolved paths directly to each sub-agent's launch prompt. Sub-agents receive the path and load the skill directly — they do NOT read this registry.

## User Skills

| Trigger | Skill | Path |
|---------|-------|------|
| SQL, SELECT *, COUNT(*), queries, migrations, repositories, ORMs, indexes, joins | sql-review | /home/platorre/.config/opencode/skills/sql-review/SKILL.md |
| "vault", "notas", "Obsidian", "actualiza nota", "guarda en stas", "documenta requerimiento", "marca estado QA", sync notes with Engram | vault-notes | /home/platorre/.config/opencode/skills/vault-notes/SKILL.md |
| Vue.js / Nuxt features, VueUse composables | vueuse-functions | /home/platorre/.config/opencode/skills/vueuse-functions/SKILL.md |

## Project Conventions

| File | Path | Notes |
|------|------|-------|
| AGENTS.md | /home/platorre/.config/opencode/AGENTS.md | Index — code style (§1), voz answer-first opt-out (§2), shrink-patterns (§3) |
| opencode.json | /home/platorre/.config/opencode/opencode.json | Agents, permissions, MCP engram, providers ollama/llama.cpp |

Read the convention files listed above for project-specific patterns and rules. All referenced paths have been extracted — no need to read index files to discover more.
