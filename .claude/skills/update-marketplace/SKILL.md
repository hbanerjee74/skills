---
name: update-marketplace
description: >
  Updates .claude-plugin/marketplace.json by scanning the repo for plugins,
  then reconciling the file with what is actually present. Validates the
  marketplace structure against the official Claude Code schema and flags
  errors in foldering or JSON. Use when a new plugin has been added, when a
  plugin has been renamed or moved, or when marketplace.json may be out of
  sync. Also use when asked to "sync the marketplace", "refresh
  marketplace.json", or "validate the marketplace".
tools: Read, Write, Glob, Grep, Bash
type: skill-builder
domain: Skill Builder
version: 2.0.0
---

# Update Marketplace

Procedure for keeping `.claude-plugin/marketplace.json` in sync with what is
actually in the repo. Scans for plugins, validates the marketplace structure
against the official Claude Code schema, and reconciles the entry list.

## Key Concept — Skills Are Auto-Discovered

Skills in `skills/` directories are **auto-discovered** by Claude Code when a
plugin is installed. They do **not** need individual marketplace entries.

In this repo, the root `plugin.json` at `.claude-plugin/plugin.json` makes the
entire repository a plugin (`"source": "./"`). All skills under `skills/` are
automatically available when the `vibedata` plugin is installed. Similarly,
skills inside `plugins/*/skills/` belong to that plugin and are auto-discovered.

**Only plugins get marketplace entries.** Never create separate entries for
individual skills.

## Quick Reference

| What gets an entry | Detected by | Entry format |
|---|---|---|
| Plugin in `plugins/` | `plugins/<dir>/.claude-plugin/plugin.json` exists | `{ "name", "description", "source" }` |
| Root plugin | `.claude-plugin/plugin.json` exists at repo root | `{ "name", "description", "source": "./" }` |

| What does NOT get an entry | Why |
|---|---|
| Skills in `skills/` | Auto-discovered via root plugin |
| Skills in `plugins/*/skills/` | Auto-discovered via their parent plugin |

Merge key: `source` path (e.g. `"./plugins/skill-builder"` or `"./"`).

## Procedure

### Step 1 — Read and validate marketplace.json

```
Read .claude-plugin/marketplace.json
```

Validate the top-level structure against the required schema:

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | Yes | Kebab-case, no spaces. Users see this when installing (`@marketplace-name`). |
| `owner` | object | Yes | Must contain `name` (string). Optional: `email` (string). |
| `plugins` | array | Yes | List of plugin entries. |
| `metadata.description` | string | No | Brief marketplace description. |
| `metadata.version` | string | No | Marketplace version. |
| `metadata.pluginRoot` | string | No | Base directory prepended to relative source paths. |

**Reserved names** (cannot be used): `claude-code-marketplace`, `claude-code-plugins`,
`claude-plugins-official`, `anthropic-marketplace`, `anthropic-plugins`, `agent-skills`,
`life-sciences`. Names impersonating official marketplaces are also blocked.

Store: top-level `name`, `owner`, `metadata` (never modify these). Index existing
`plugins` entries by their `source` value for the merge in Step 4.

**If validation fails**, report the errors and stop. Do not proceed to merge.

### Step 2 — Discover plugins in `plugins/`

Glob `plugins/*/.claude-plugin/plugin.json`. For each match:

- Read the JSON file.
- Extract `name` and `description`.
- Derive `source` as `"./plugins/<dir-name>"` where `<dir-name>` is the
  immediate parent of `.claude-plugin/`.

### Step 3 — Check root plugin entry

Check whether `.claude-plugin/plugin.json` exists at the repo root.

- If it exists, read it and extract `name` and `description`.
- This entry uses `"source": "./"`.
- If it does NOT exist but the current marketplace.json has a `"source": "./"`
  entry, flag it as an error: the root plugin.json is missing.

### Step 4 — Merge

For each discovered source (plugins from `plugins/` alphabetical by name, then
root plugin entry last):

| Case | Action |
|---|---|
| Source exists in repo AND in current marketplace.json | Update `name` and `description` from the source file. Preserve any other fields on the existing entry (e.g. manually-set `version`). |
| Source exists in repo but NOT in marketplace.json | **Do not silently add.** Flag the entry, print a warning with the proposed entry, and ask the user whether to add it before writing. |
| Entry is in marketplace.json but source directory does not exist | **Do not silently remove.** Flag the entry, print a warning, and ask the user whether to remove it before writing. |

### Step 5 — Validate entries

Before writing, validate every entry in the merged list:

**Required per entry:**

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | Yes | Kebab-case, no spaces. |
| `source` | string or object | Yes | Relative paths must start with `./`. No `..` path traversal. |

**Optional per entry:**

| Field | Type | Notes |
|---|---|---|
| `description` | string | Brief plugin description. |
| `version` | string | Semver. Avoid setting in both marketplace entry AND `plugin.json` — manifest always wins silently. |
| `author` | object | `name` required, `email` optional. |
| `homepage` | string | Documentation URL. |
| `repository` | string | Source code URL. |
| `license` | string | SPDX identifier (e.g. `MIT`, `Apache-2.0`). |
| `keywords` | array | Discovery tags. |
| `category` | string | Plugin category. |
| `tags` | array | Searchability tags. |
| `strict` | boolean | Default `true`. Only set `false` when marketplace entry is the entire definition. |

**Validation checks:**

- No duplicate `name` values across all entries.
- All `source` strings start with `./` (no absolute paths, no `..`).
- No entries pointing to `skills/` directories — skills are auto-discovered.
- If `strict` is omitted or `true`, verify the source directory has `.claude-plugin/plugin.json`.
- If `strict: false`, verify there is no `plugin.json` in the source directory (conflict would cause load failure).

**If validation finds errors**, report them and ask before writing.

### Step 6 — Write

Overwrite `.claude-plugin/marketplace.json` with the merged result:

- Preserve `name`, `owner`, `metadata` at the top level exactly as read.
- Write the `plugins` array sorted alphabetically by `name`.
- Format as pretty-printed JSON with 2-space indentation.
- Compact entries with short values onto one line for readability
  (e.g. `{ "name": "skill-builder", "description": "...", "source": "./plugins/skill-builder" }`).

### Step 7 — Report

After writing, print a summary:

```
marketplace.json updated
  Added:   <n> entries — <names> (user confirmed)
  Pending: <n> entries awaiting add confirmation — <names>
  Updated: <n> entries — <names>
  Flagged: <n> entries with missing source — <names> (awaiting confirmation)
  Errors:  <n> validation errors — <details>
  Unchanged: <n> entries
```

## Entry format reference

```json
// Plugin in plugins/ — no strict field (defaults true)
{
  "name": "skill-builder",
  "description": "Multi-agent workflow for creating domain-specific Claude skills",
  "source": "./plugins/skill-builder"
}

// Root plugin — entire repo as a plugin, skills auto-discovered
{
  "name": "vibedata",
  "description": "Practitioner-level data and analytics engineering skills for Claude",
  "source": "./"
}
```

## Validation error reference

| Error | Cause | Fix |
|---|---|---|
| Missing `owner` or `owner.name` | Top-level `owner` object is required | Add `"owner": { "name": "..." }` |
| Missing `plugins` array | Top-level `plugins` is required | Add `"plugins": []` |
| Duplicate plugin name `"X"` | Two entries share the same name | Give each plugin a unique name |
| Source path contains `..` | Path traversal not allowed | Use paths relative to repo root without `..` |
| Source does not start with `./` | Relative paths must start with `./` | Prefix with `./` |
| Entry for `skills/X` found | Skills are auto-discovered, not listed | Remove the entry — skills are found via the parent plugin |
| `strict: true` but no `plugin.json` | Plugin needs a manifest when strict | Add `.claude-plugin/plugin.json` or set `"strict": false` |
| `strict: false` with `plugin.json` declaring components | Conflict — plugin would fail to load | Remove components from `plugin.json` or remove `"strict": false` |
| `source: "./"` but no root `plugin.json` | Root entry requires `.claude-plugin/plugin.json` | Create the file or remove the entry |
| Version set in both marketplace entry and `plugin.json` | Manifest always wins silently | Set version in only one place |
| Reserved marketplace name | Name is reserved by Anthropic | Choose a different marketplace name |

## Common mistakes

- **Creating entries for individual skills** — skills in `skills/` are auto-discovered via the root plugin. Only plugins get marketplace entries.
- **Adding `strict: false` to plugin entries that have `plugin.json`** — causes a conflict if `plugin.json` declares any components.
- **Using directory name as `name` when `plugin.json` has a different name** — always prefer the `name` from `plugin.json`.
- **Scanning `plugins/*/skills/`** — those skills belong to the plugin, not the marketplace.
- **Modifying `name`, `owner`, or `metadata`** — these are set once and not derived from plugins.
- **Silently removing stale entries** — always flag and confirm before deleting.
- **Setting version in both marketplace entry and `plugin.json`** — manifest always wins. Set in one place only.
- **Using absolute paths or `..` in source** — all sources must be relative, starting with `./`.

## References

See `references/evaluations.md` for test scenarios.
