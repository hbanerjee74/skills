# Evaluations: update-marketplace

### Scenario 1: New skill added to repo

**Prompt**: "I just added a new skill at `skills/crm-domain/`. Can you update the marketplace?"

**Expected behavior**: Claude globs `skills/*/SKILL.md`, finds `skills/crm-domain/SKILL.md`, reads front matter for `name` and `description`, checks that `./skills/crm-domain` is not already in marketplace.json, appends a new entry with `strict: false`, writes the file, reports "Added: 1 entry — crm-domain".

**Pass criteria**:
- New entry appears in `plugins` array with `"strict": false` and correct `source`
- `name` and `description` match what is in the SKILL.md front matter (not the directory name)
- All pre-existing entries are unchanged

---

### Scenario 2: Skill renamed / directory moved

**Prompt**: "I renamed the `cat-food` skill directory to `mock-data`. Sync the marketplace."

**Expected behavior**: Claude discovers `./skills/mock-data` (new, not in marketplace.json) and detects that `./skills/cat-food` (existing entry) no longer has a directory. It flags the stale `cat-food` entry and asks for confirmation before removing it. After confirmation, removes the old entry and adds the new one. Reports "Added: 1 — mock-data, Flagged: 1 — mock-skill (source ./skills/cat-food not found)".

**Pass criteria**:
- Claude does NOT silently delete the stale entry without asking
- After confirmation, the old entry is removed and the new entry is written
- `name` on the new entry comes from the SKILL.md front matter, not the directory name

---

### Scenario 3: Plugin description updated

**Prompt**: "The description in `plugins/skill-builder/.claude-plugin/plugin.json` was updated. Refresh marketplace.json."

**Expected behavior**: Claude reads the updated `plugin.json`, finds the existing `skill-builder` entry in marketplace.json (matched by source `./plugins/skill-builder`), and updates `name` and `description` to match. Does not add a `description` field if one wasn't there before (plugin entries in marketplace.json typically omit description since it comes from plugin.json).

**Pass criteria**:
- Only the `skill-builder` entry is changed
- `strict` is not added to the plugin entry
- Report shows "Updated: 1 — skill-builder"

---

### Scenario 4: Full sync on clean repo

**Prompt**: "Sync marketplace.json from scratch — assume all entries may be stale."

**Expected behavior**: Claude discovers all 4 plugins in `plugins/` and all skill directories in `skills/`, reads each source file, builds the merged entry list. All existing entries have matching source directories so no entries are flagged. The file is rewritten with plugins first (alphabetical), then skills (alphabetical). Report shows counts for updated/unchanged entries.

**Pass criteria**:
- All discovered sources appear in the output
- Plugins precede standalone skills in the `plugins` array
- Within each group, entries are sorted alphabetically by `name`
- Top-level `name`, `owner`, `metadata` are unchanged
