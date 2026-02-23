Update `.claude-plugin/marketplace.json` by scanning the repo and reconciling it with what is actually present. Follow this procedure:

**Step 1 — Read current state**
Read `.claude-plugin/marketplace.json`. Store the top-level `name`, `owner`, and `metadata` — never modify these. Index existing `plugins` entries by their `source` value.

**Step 2 — Discover plugins**
Glob `plugins/*/\.claude-plugin/plugin.json`. For each match, read `name` and `description` from the JSON. Derive `source` as `"./plugins/<dir-name>"`.

**Step 3 — Discover standalone skills**
Glob `skills/*/SKILL.md`. For each match, parse the YAML front matter and extract `name` and `description`. Fall back to directory name if `name` is missing. Derive `source` as `"./skills/<dir-name>"`. Do not recurse into `plugins/*/skills/`.

**Step 4 — Merge (keyed on `source`)**
- Source exists in repo AND in marketplace.json → update `name` and `description` from source file; preserve other existing fields
- Source exists in repo but NOT in marketplace.json → add new entry
- Entry in marketplace.json but source directory missing → flag it, do not silently remove; ask for confirmation

**Step 5 — Write**
Overwrite `.claude-plugin/marketplace.json`:
- Preserve top-level `name`, `owner`, `metadata`
- Plugin entries first, then standalone skill entries; alphabetical by `name` within each group
- Plugin entry format: `{ "name": "...", "source": "./plugins/<dir>" }` — no `strict` field
- Standalone skill entry format: `{ "name": "...", "description": "...", "source": "./skills/<dir>", "strict": false }`
- Pretty-print with 2-space indentation

**Step 6 — Report**
Print a summary: entries added, updated, flagged, unchanged.
