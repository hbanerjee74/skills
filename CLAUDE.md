# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A public marketplace for [Vibedata](https://acceleratedata.ai) hosting two types of content:

- **Standalone skills** (`skills/`) — Markdown knowledge packages injected into Vibedata agent system prompts at runtime
- **Agent plugins** (`plugins/`) — Full Claude Code plugins with agents, skills, and multi-step workflows distributed via the marketplace

There are no build scripts or test runners — this is a content repository. Changes are Markdown edits.

---

## Standalone skills (`skills/`)

Each skill lives under `skills/` in a directory whose name must **exactly match** the `name` field in its `SKILL.md` front matter.

```
skills/
└── <skill-name>/
    ├── SKILL.md            # Required — skill content and front matter
    ├── references/         # Optional — supporting files loaded alongside the skill
    └── context/            # Optional — process artifacts (research, decisions, evals in progress)
```

### Required front matter fields

Every `SKILL.md` must open with a YAML front matter block:

```yaml
---
name: dbt-fabric-patterns          # kebab-case, max 64 chars, matches directory name
description: >                     # Format: "[What]. Use when [triggers]. [How]. Also use when [more triggers]."
  ...                              # Max 1024 chars. Third person — injected into system prompt.
tools: Read, Write, Edit, Glob, Grep, Bash
type: platform                     # One of: platform | domain | source | data-engineering | skill-builder
domain: dbt on Microsoft Fabric    # Business or technical domain this skill covers
version: 1.0.0                     # Semantic version — increment on meaningful updates
---
```

### Skill types and where they appear in Vibedata

| Type | Appears in |
|---|---|
| `platform`, `domain`, `source`, `data-engineering` | Skill Library (starting points for building new skills) |
| `skill-builder` | Settings → Skills (active for every agent session) |

### Content rules

Skills must contain **hard-to-find practitioner knowledge** that Claude cannot get from Context7 or its training data. Test: "Would Claude get this from Context7 + its training data?" If yes, omit it.

Good content: undocumented platform quirks, decision frameworks for choosing between approaches, anti-patterns from real-world use, cross-tool integration patterns.

Not suitable: rehash of official docs, generic best practices, single-company private setup details.

### SKILL.md constraints

- **Under 500 lines** — if a section grows past a few paragraphs, extract it to `references/`
- Reference files are **one level deep only** (no nesting inside `references/`)
- Files over 100 lines need a table of contents
- No `context/` files belong in `references/` — process artifacts (clarifications, decisions, research plans, validation logs) go in `context/` only

### Required sections (SKILL.md body)

- **Overview** — scope, audience, key concepts. No restatement of the description front matter.
- **Quick reference** — top patterns most likely to prevent mistakes
- **Pointers to references** — what each `references/` file covers and when to read it

For **platform** and **data-engineering** skills: add a **Getting Started** checklist (5–8 ordered steps) after Quick Reference.

For **source** and **domain** skills: no Getting Started section — sections are parallel and order-independent.

### Evaluations

Every skill should include `references/evaluations.md` with at least 3 runnable scenarios:

```
### Scenario N: [Short name]
**Prompt**: [Exact prompt to send to Claude with this skill active]
**Expected behavior**: [What Claude should do — specific, observable]
**Pass criteria**: [1-2 measurable signals the skill is working]
```

### Description field triggers

Trigger conditions belong **exclusively in the description front matter**. Do NOT add a "When to Use This Skill" section in the body — it duplicates the description and wastes context budget.

For dbt skills, include layer-specific triggers so the skill activates on silver/gold work:
> `Also use when the user mentions "[domain] models", "silver layer", "gold layer", "marts", or "[domain]-specific dbt".`

---

## Agent plugins (`plugins/`)

Plugins are full Claude Code extensions with `agents/`, `skills/`, and a `plugin.json` manifest. Based on the [official Claude Code docs](https://code.claude.com/docs/en/plugins).

### Plugin directory structure

```
plugins/
└── <plugin-name>/
    ├── .claude-plugin/
    │   └── plugin.json     # Required manifest — ONLY this file goes in .claude-plugin/
    ├── agents/             # Agent definition .md files
    ├── skills/             # Agent skills, each in their own subdirectory
    │   └── <skill-name>/
    │       └── SKILL.md
    ├── commands/           # Slash command .md files
    ├── hooks/              # hooks.json for event handlers
    ├── .mcp.json           # MCP server configurations
    └── settings.json       # Default settings applied when plugin is enabled
```

> **Common mistake**: Do not put `agents/`, `skills/`, `commands/`, or `hooks/` inside `.claude-plugin/`. Only `plugin.json` goes there.

### `plugin.json` schema

Based on the [official spec](https://code.claude.com/docs/en/plugins-reference).

**Required fields:**

| Field | Description |
|---|---|
| `name` | Plugin identifier and skill namespace. Skills in the plugin are invoked as `/<name>:<skill>`. Kebab-case, no spaces. |
| `version` | Semantic version (e.g. `1.0.0`). Must change between releases for update detection to work. |
| `description` | Shown in the plugin manager when users browse or install. |

**Optional fields:**

| Field | Description |
|---|---|
| `author.name` | Plugin author name |
| `author.email` | Contact email |
| `homepage` | Documentation URL |
| `repository` | Source code URL |
| `license` | SPDX identifier (e.g. `MIT`, `ELv2`) |
| `keywords` | Array of strings for discoverability |

**Do not add** a `skills` field to `plugin.json` — it is not in the official spec. Skills are auto-discovered from the `skills/` directory at the plugin root.

### Skills inside plugins

A skill inside a plugin follows the same `SKILL.md` structure as standalone skills, but with a simpler frontmatter requirement. The minimum required fields are `name` and `description`:

```yaml
---
name: skill-name          # Becomes /<plugin-name>:<skill-name> when installed
description: >
  What this skill does. Use when [triggers].
---
```

Vibedata-specific fields (`type`, `domain`, `tools`, `version`, `trigger`) can be added when the skill is also published as a standalone skill in `skills/`.

### Testing plugins locally

```bash
# Load a plugin without installing it
claude --plugin-dir ./plugins/skill-builder

# Try a skill from the loaded plugin
/skill-builder:building-skills

# Load multiple plugins at once
claude --plugin-dir ./plugins/skill-builder --plugin-dir ./plugins/skill-builder-research

# Validate plugin structure
claude plugin validate ./plugins/skill-builder
```

---

## `.claude-plugin/marketplace.json` format

Based on the [official Claude Code docs](https://code.claude.com/docs/en/plugin-marketplaces).

### Marketplace-level fields

| Field | Required | Description |
|---|---|---|
| `name` | Yes | Marketplace identifier — kebab-case, no spaces. Users see this when installing: `/plugin install tool@<name>`. Reserved names (e.g. `agent-skills`) cannot be used. |
| `owner.name` | Yes | Maintainer name |
| `owner.email` | No | Contact email |
| `metadata.description` | No | Brief marketplace description |
| `metadata.version` | No | Marketplace version |
| `metadata.pluginRoot` | No | Base directory prepended to relative `source` paths |

### Plugin entry fields

Each entry in `plugins` requires `name` and `source`. All other fields are optional.

| Field | Description |
|---|---|
| `name` | Plugin identifier — kebab-case, no spaces |
| `source` | Where to fetch the plugin. Use `"./path"` for local directories (must start with `./`). Also supports `{ "source": "github", "repo": "owner/repo" }`, npm, pip, and git URL sources. |
| `description` | Brief plugin description |
| `version` | Plugin version |
| `strict` | `true` (default): `plugin.json` is the authority. `false`: marketplace entry is the complete definition — use for skill-only entries with no `plugin.json`. |
| `agents` | Custom paths to agent files |
| `commands` | Custom paths to command files or directories |
| `hooks` | Hooks configuration or path to hooks file |
| `mcpServers` | MCP server configurations |

**Notes:**
- Skills are auto-discovered from `SKILL.md` files in the plugin directory — there is no `skills` field in `marketplace.json` or `plugin.json`.
- Do not add a `$schema` field — it is not part of the documented spec.
- Standalone skill entries (no `plugin.json`) use `"strict": false`.

### Validate and test

```bash
# Validate marketplace.json syntax and structure
claude plugin validate .

# Test by adding the marketplace locally
/plugin marketplace add ./

# Install a specific plugin to verify it works
/plugin install skill-builder@vibedata-skills
/plugin install dbt-fabric-patterns@vibedata-skills
```

---

## Anti-patterns

### Standalone skills
- Nested reference files — keep one level deep from `SKILL.md`
- "When to Use This Skill" body section — triggers belong in description frontmatter only
- "Questions for your stakeholder" blocks — stakeholder communication belongs in `context/decisions.md`
- Process artifacts in the skill output directory (clarifications.md, decisions.md, research-plan.md, validation logs)
- `dbt-utils` macros on Fabric — use `tsql-utils` instead
- Mixing `dlt` (dlthub) with Databricks DLT terminology
- Vague descriptions like "configure your data warehouse"
- Windows paths — always use forward slashes

### Plugins
- Files other than `plugin.json` inside `.claude-plugin/` — agents, skills, commands go at the plugin root
- `"skills"` field in `plugin.json` — not in the spec; skills are auto-discovered
- Single-line plugin.json — format as pretty-printed JSON for maintainability
- Bumping `version` in both `plugin.json` and the marketplace entry — the manifest always wins; set version in one place only
