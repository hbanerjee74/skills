# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A public skills marketplace for [Vibedata](https://accleratedata.ai). Skills are Markdown knowledge packages that teach Claude domain-specific patterns for data and analytics engineering. They are imported into Vibedata and injected into agent system prompts at runtime.

There are no build scripts or test runners — this is a content repository. Changes are Markdown edits.

## Skill structure

Each skill lives under `skills/` in a directory whose name must **exactly match** the `name` field in its `SKILL.md` front matter.

```
skills/
└── <skill-name>/
    ├── SKILL.md            # Required — skill content and front matter
    ├── references/         # Optional — supporting files loaded alongside the skill
    └── context/            # Optional — process artifacts (research, decisions, evals in progress)
```

The `.claude-plugin/marketplace.json` file registers all plugins and skills so Claude Code can discover them as a plugin marketplace (see [marketplace.json format](#claude-pluginmarketplacejson-format) below).

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

## Content rules

Skills must contain **hard-to-find practitioner knowledge** that Claude cannot get from Context7 or its training data. Test: "Would Claude get this from Context7 + its training data?" If yes, omit it.

Good content: undocumented platform quirks, decision frameworks for choosing between approaches, anti-patterns from real-world use, cross-tool integration patterns.

Not suitable: rehash of official docs, generic best practices, single-company private setup details.

### SKILL.md constraints

- **Under 500 lines** — if a section grows past a few paragraphs, extract it to `references/`
- Reference files are one level deep only (no nesting inside `references/`)
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

## Description field triggers

Trigger conditions belong **exclusively in the description front matter**. Do NOT add a "When to Use This Skill" section in the body — it duplicates the description and wastes context budget.

For dbt skills, include layer-specific triggers so the skill activates on silver/gold work:
> `Also use when the user mentions "[domain] models", "silver layer", "gold layer", "marts", or "[domain]-specific dbt".`

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
| `metadata.pluginRoot` | No | Base directory prepended to relative `source` paths (e.g. `"./plugins"` lets you write `"source": "formatter"` instead of `"source": "./plugins/formatter"`) |

### Plugin entry fields

Each entry in `plugins` requires `name` and `source`. All other fields are optional.

| Field | Description |
|---|---|
| `name` | Plugin identifier — kebab-case, no spaces |
| `source` | Where to fetch the plugin. Use `"./path"` for local directories within the repo (must start with `./`). Also supports `{ "source": "github", "repo": "owner/repo" }`, npm, pip, and git URL sources. |
| `description` | Brief plugin description |
| `version` | Plugin version |
| `strict` | `true` (default): `plugin.json` inside the plugin directory is the authority. `false`: marketplace entry is the complete definition — use this for plugins that have no `plugin.json`. |
| `commands` | Custom paths to command files or directories |
| `agents` | Custom paths to agent files |
| `hooks` | Hooks configuration or path to hooks file |
| `mcpServers` | MCP server configurations |
| `lspServers` | LSP server configurations |

**Note:** Skills are auto-discovered by Claude Code when it scans the plugin directory for `SKILL.md` files. There is no `skills` field in `marketplace.json`. Do not add a `$schema` field — it is not part of the documented spec.

Skill-only entries (no `plugin.json` in the directory) should use `"strict": false` so Claude Code treats the marketplace entry as the complete plugin definition.

### Validate and test

```bash
# Validate marketplace.json syntax and structure
claude plugin validate .

# Test by adding the marketplace locally
/plugin marketplace add ./

# Install a plugin to verify it works
/plugin install dbt-fabric-patterns@vibedata-skills
```

## Anti-patterns

- Nested reference files — keep one level deep from `SKILL.md`
- Windows paths — always use forward slashes
- "When to Use This Skill" body section — triggers belong in description frontmatter only
- "Questions for your stakeholder" blocks — stakeholder communication belongs in `context/decisions.md`
- Process artifacts in the skill output directory (clarifications.md, decisions.md, research-plan.md, validation logs)
- `dbt-utils` macros on Fabric — use `tsql-utils` instead
- Mixing `dlt` (dlthub) with Databricks DLT terminology
- Vague descriptions like "configure your data warehouse"
