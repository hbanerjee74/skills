---
name: update-marketplace
description: >
  Updates .claude-plugin/marketplace.json by scanning the repo for plugins and
  standalone skills, then reconciling the file with what is actually present.
  Use when a new skill or plugin has been added to the repo, when a skill has
  been renamed or its directory moved, or when marketplace.json may be out of
  sync. Reads plugin.json and SKILL.md front matter to extract names and
  descriptions automatically. Also use when asked to "sync the marketplace",
  "refresh marketplace.json", or "add a new skill to the marketplace".
tools: Read, Write, Glob, Grep, Bash
type: skill-builder
domain: Skill Builder
version: 1.0.0
---

# Update Marketplace

Procedure for keeping `.claude-plugin/marketplace.json` in sync with what is
actually in the repo. Scans `plugins/` and `skills/`, reads metadata from
source files, and reconciles with the existing marketplace entry list.

## Quick Reference

| Source type | Detected by | `strict` | Entry format |
|---|---|---|---|
| Plugin | `plugins/<dir>/.claude-plugin/plugin.json` exists | omit (defaults `true`) | `{ "name", "source" }` |
| Standalone skill | `skills/<dir>/SKILL.md` exists | `false` | `{ "name", "description", "source", "strict": false }` |

Merge key: `source` path (e.g. `"./plugins/skill-builder"` or `"./skills/dbt-fabric-patterns"`).

## Procedure

### Step 1 — Read the current marketplace.json

```
Read .claude-plugin/marketplace.json
```

Store: top-level `name`, `owner`, `metadata` (never modify these). Index existing
`plugins` entries by their `source` value for the merge in Step 4.

### Step 2 — Discover plugins

Glob `plugins/*/\.claude-plugin/plugin.json`. For each match:

- Read the JSON file.
- Extract `name` and `description`.
- Derive `source` as `"./plugins/<dir-name>"` where `<dir-name>` is the
  immediate parent of `.claude-plugin/`.

### Step 3 — Discover standalone skills

Glob `skills/*/SKILL.md`. For each match:

- Parse the YAML front matter block (between the opening and closing `---`).
- Extract `name` and `description`.
- If `name` is missing from front matter, fall back to the directory name.
- If `description` is missing, omit the `description` field from the entry.
- Derive `source` as `"./skills/<dir-name>"`.

> **Do not recurse into `plugins/*/skills/`** — skills inside agent plugins
> are not listed in the top-level marketplace.json.

### Step 4 — Merge

For each discovered source (plugins first, then skills, both alphabetical by name):

| Case | Action |
|---|---|
| Source exists in repo AND in current marketplace.json | Update `name` and `description` from the source file. Preserve any other fields on the existing entry (e.g. manually-set `version`). |
| Source exists in repo but NOT in marketplace.json | **Do not silently add.** Flag the entry, print a warning with the proposed entry, and ask the user whether to add it before writing. |
| Entry is in marketplace.json but source directory does not exist | **Do not silently remove.** Flag the entry, print a warning, and ask the user whether to remove it before writing. |

### Step 5 — Write

Overwrite `.claude-plugin/marketplace.json` with the merged result:

- Preserve `name`, `owner`, `metadata` at the top level exactly as read.
- Write the `plugins` array: plugin entries first, then standalone skill entries.
- Within each group, sort alphabetically by `name`.
- Format as pretty-printed JSON with 2-space indentation.
- Compact single-field plugin entries onto one line for readability
  (e.g. `{ "name": "skill-builder", "source": "./plugins/skill-builder" }`).

### Step 6 — Report

After writing, print a summary:

```
marketplace.json updated
  Added:   <n> entries — <names> (user confirmed)
  Pending: <n> entries awaiting add confirmation — <names>
  Updated: <n> entries — <names>
  Flagged: <n> entries with missing source — <names> (awaiting confirmation)
  Unchanged: <n> entries
```

## Entry format reference

```json
// Plugin entry — no strict field
{ "name": "skill-builder", "source": "./plugins/skill-builder" }

// Standalone skill entry — strict: false required
{
  "name": "dbt-fabric-patterns",
  "description": "Practitioner-level dbt patterns...",
  "source": "./skills/dbt-fabric-patterns",
  "strict": false
}
```

## Common mistakes

- **Adding `strict: false` to plugin entries** — only standalone skills (no `plugin.json`) need `strict: false`.
- **Using directory name as `name` when front matter has a different name** — always prefer the `name` field from `plugin.json` or SKILL.md front matter.
- **Scanning `plugins/*/skills/`** — those skills belong to the plugin, not the marketplace top-level listing.
- **Modifying `name`, `owner`, or `metadata`** — these are set once and not derived from individual skill files.
- **Silently removing stale entries** — always flag and confirm before deleting.

## References

See `references/evaluations.md` for test scenarios.
