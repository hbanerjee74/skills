# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

A public marketplace for [Vibedata](https://acceleratedata.ai) hosting:

- **Standalone skills** (`skills/`) — Markdown knowledge packages loaded into Vibedata agents at runtime
- **Agent plugins** (`plugins/`) — Full Claude Code plugins with agents, skills, and multi-step workflows

No build scripts or test runners. All changes are Markdown or JSON edits.

---

## Standalone skills (`skills/`)

Directory name must **exactly match** the `name` field in `SKILL.md` front matter.

```
skills/
└── <skill-name>/
    ├── SKILL.md        # Required
    ├── references/     # Supporting files, one level deep only
    └── context/        # Process artifacts only (never published)
```

### Front matter

```yaml
---
name: dbt-fabric-patterns          # kebab-case, matches directory name
description: >                     # "[What]. Use when [triggers]." Third person. Max 1024 chars.
  ...
tools: Read, Write, Edit, Glob, Grep, Bash
type: platform                     # platform | domain | source | data-engineering | skill-builder
domain: dbt on Microsoft Fabric
version: 1.0.0
---
```

| Type | Appears in Vibedata |
|---|---|
| `platform`, `domain`, `source`, `data-engineering` | Skill Library |
| `skill-builder` | Settings → Skills (active every session) |

### Structural constraints

- `SKILL.md` under 500 lines — extract long sections to `references/`
- `references/` is one level deep — no subdirectory nesting
- Files over 100 lines need a table of contents
- `context/` is for process artifacts only (clarifications, decisions, research) — never referenced from `SKILL.md`

---

## Agent plugins (`plugins/`)

```
plugins/
└── <plugin-name>/
    ├── .claude-plugin/
    │   └── plugin.json     # ONLY this file goes in .claude-plugin/
    ├── agents/
    ├── skills/<skill-name>/SKILL.md
    ├── commands/
    └── hooks/
```

`plugin.json` required fields: `name`, `version`, `description`. Optional: `author`, `homepage`, `repository`, `license`, `keywords`. Full spec: [code.claude.com/docs/en/plugins-reference](https://code.claude.com/docs/en/plugins-reference).

Do not add a `skills` field — skills are auto-discovered from the `skills/` directory.

### Testing

```bash
claude --plugin-dir ./plugins/skill-builder      # load without installing
claude plugin validate ./plugins/skill-builder   # validate structure
```

---

## `.claude-plugin/marketplace.json`

- Plugin entries: `{ "name", "source": "./plugins/<dir>" }` — no `strict` field needed
- Standalone skill entries: add `"description"` and `"strict": false`
- Skills are auto-discovered — no `skills` field in entries, no `$schema`

To sync this file after adding or moving skills, use the **`update-marketplace`** skill (`.claude/skills/update-marketplace/`).

---

## Anti-patterns

**Skills**
- Nested `references/` subdirectories — one level only
- "When to Use This Skill" body section — triggers belong in `description` frontmatter only
- Process artifacts in `references/` — they go in `context/` only
- `dbt-utils` macros on Fabric — use `tsql-utils` instead
- Mixing `dlt` (dlthub) with Databricks DLT terminology

**Plugins**
- Files other than `plugin.json` inside `.claude-plugin/`
- `"skills"` field in `plugin.json` — not in spec, skills are auto-discovered
- Single-line `plugin.json` — format as pretty-printed JSON
- Bumping `version` in both `plugin.json` and the marketplace entry — manifest always wins
