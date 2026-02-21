# Vibedata Skills Marketplace

A public collection of skills for [Vibedata](https://accleratedata.ai) — structured knowledge packages that teach Claude domain-specific patterns for data and analytics engineering.

---

## What are Vibedata Skills?

Vibedata Skills are [Claude agent skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) — curated, practitioner-level knowledge packages for data and analytics engineering. Each skill teaches Claude domain-specific patterns for platforms, source systems, business domains, and engineering practices.

Skills published here can be imported into Vibedata directly from this repository.

---

## Skill Marketplace

This repository is the official Vibedata Skills Marketplace. Build a skill using [Vibedata](https://accleratedata.ai), then share it with the community by submitting a pull request. Once merged, your skill is available for anyone to import directly from the app.

### Where skills appear in the app

| Skill type | Where it's imported |
|---|---|
| `platform`, `domain`, `source`, `data-engineering` | **Skill Library** — used as starting points or references when building new skills with the multi-agent workflow |
| `skill-builder` | **Settings → Skills** — loaded directly into Vibedata's agent as active domain knowledge |

### Importing from the marketplace

**Skill Library (in the Vibedata app):**
1. Open the Skill Library and click **Import from Marketplace**
2. Browse skills filtered by type — platform, domain, source, or data-engineering
3. Select one or more skills and import
4. Imported skills are available in refine-only mode

**Settings → Skills:**
1. Go to **Settings → Skills** and click **Import from Marketplace**
2. Only `skill-builder` type skills are shown
3. Active skills are injected into the agent's instructions for every session

### Configuring the marketplace URL

The marketplace URL defaults to this repository (`https://github.com/hbanerjee74/skills`) and can be changed in **Settings → Marketplace URL** to point to a private or team repository.

---

## Skill Types

| Type | Description | Appears in |
|---|---|---|
| `platform` | Tools and platform-specific skills — dbt, Microsoft Fabric, Databricks | Skill Library |
| `domain` | Business domain knowledge — Finance, Marketing, HR, Revenue | Skill Library |
| `source` | Source system extraction patterns — Salesforce, SAP, Workday | Skill Library |
| `data-engineering` | Technical patterns and practices — SCD, Incremental Loads, Medallion | Skill Library |
| `skill-builder` | Skills for use with the Skill Builder app itself | Settings → Skills |

---

## Required Front Matter

Every skill's `SKILL.md` must include the following front matter fields:

### Standard Claude Code fields

| Field | Required | Description |
|---|---|---|
| `name` | Yes | Unique skill identifier. Kebab-case, lowercase, max 64 chars (e.g. `dbt-fabric-patterns`) |
| `description` | Yes | What the skill covers and when Claude should load it. Format: `[What it does]. Use when [triggers]. [How it works].` Max 1024 chars. |
| `tools` | Yes | Comma-separated list of Claude tools the skill may invoke (e.g. `Read, Write, Edit, Glob, Grep, Bash`) |

### Vibedata-specific fields

| Field | Required | Description |
|---|---|---|
| `type` | Yes | One of the 5 types in the table above |
| `version` | Yes | Semantic version (e.g. `1.0.0`). Increment on meaningful updates. |

### Example

```yaml
---
name: dbt-fabric-patterns
description: >
  Build dbt models on Microsoft Fabric. Use when designing materializations,
  incremental strategies, or snapshots on Fabric. Also use when the user
  mentions OneLake, lakehouse, or Fabric-specific dbt quirks.
tools: Read, Write, Edit, Glob, Grep, Bash
type: platform
version: 1.0.0
---
```

---

## Folder Structure

Each skill lives in its own root-level directory named after the skill. The directory name must match the `name` field in the front matter.

```
<skill-name>/
├── SKILL.md            # Required — skill content and front matter
├── references/         # Optional — supporting reference files
│   ├── patterns.md
│   └── quirks.md
└── context/            # Optional — research context, decisions, evaluations
    └── evaluations.md
```

**Rules:**
- One skill per directory
- Directory name = `name` field in front matter (kebab-case)
- `SKILL.md` must be at the root of the skill directory
- `SKILL.md` must be under 500 lines
- Reference files go in `references/` — these are loaded alongside the skill

---

## Content Guidelines

Skills in this marketplace must contain **hard-to-find practitioner knowledge** — not content that can be retrieved from official documentation or Context7.

Good skill content:
- Undocumented platform quirks and workarounds
- Decision frameworks for choosing between approaches
- Anti-patterns discovered through real-world use
- Cross-tool integration patterns (e.g. dlt → dbt → Elementary)

Not suitable:
- Rehash of official docs
- Generic best practices available in any tutorial
- Content specific to a single company's private setup

---

## Contributing

1. Fork this repository
2. Create a directory for your skill at the repo root
3. Add `SKILL.md` with all required front matter fields
4. Add `references/` files if needed
5. Submit a pull request — include a brief description of what hard-to-find knowledge the skill captures

All submissions are reviewed before merging. Skills that fail front matter validation or contain only publicly available documentation will be requested for revision.

---

## License

MIT — see [LICENSE](./LICENSE)
