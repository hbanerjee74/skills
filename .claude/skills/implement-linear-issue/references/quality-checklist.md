# Quality Checklist

Standards from:
- [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Plugin marketplace schema](https://code.claude.com/docs/en/plugin-marketplaces)

---

## SKILL.md authoring

### Front matter
- [ ] `name`: max 64 chars, lowercase letters/numbers/hyphens only, matches directory name
- [ ] `name`: does not contain "anthropic" or "claude"
- [ ] `description`: non-empty, max 1024 chars, no XML tags
- [ ] `description`: written in **third person** ("Processes…", not "I can…" or "You can…")
- [ ] `description`: says both **what** the skill does and **when** to use it (triggers/contexts)
- [ ] `description`: specific with key terms — not vague ("Helps with documents" is bad)
- [ ] Repo-required fields present: `tools`, `type`, `domain`, `version`

### Body
- [ ] Under 500 lines
- [ ] No "When to Use This Skill" section — triggers belong in `description` only
- [ ] No time-sensitive dates or "before/after X date" language
- [ ] Consistent terminology throughout (pick one term per concept, use it everywhere)
- [ ] No Windows-style paths — use forward slashes only (`reference/guide.md` not `reference\guide.md`)
- [ ] Reference files linked from SKILL.md are **one level deep** — no chained references (`SKILL.md → a.md → b.md`)
- [ ] Reference files over 100 lines have a **table of contents**

---

## marketplace.json schema

Run `claude plugin validate .` from the repo root to catch JSON/schema errors automatically.

### Top-level fields
- [ ] `name` present (kebab-case, no spaces)
- [ ] `owner.name` present
- [ ] `plugins` array present

### Each plugin entry
- [ ] `name` present (kebab-case, no spaces) — no duplicates across entries
- [ ] `source` present; relative paths start with `./`, no `..` path traversal
- [ ] Plugin entries with their own `plugin.json`: **omit** `strict` (defaults to `true`)
- [ ] Standalone skill entries (no `plugin.json`): `"strict": false` present
- [ ] `version` not set in **both** the marketplace entry and `plugin.json` — `plugin.json` always wins silently; for relative-path plugins set it in the marketplace entry only

### plugin.json
- [ ] Pretty-printed JSON (not single-line)
- [ ] No `skills` field — skills are auto-discovered
- [ ] `name`, `version`, `description` present
