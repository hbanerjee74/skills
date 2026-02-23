# Git and PR Conventions

## PR Body Template

Title: `<issue-id>: short description`

```markdown
Fixes <issue-id>

<!-- If the PR covers child issues, list EACH on its own line: -->
<!-- Fixes VD-530 -->
<!-- Fixes VD-531 -->
<!-- Fixes VD-532 -->
<!-- NEVER group as "Fixes VD-530/531/532" — each must be a separate line. -->

## Summary
[2-3 sentences from implementation status]

## Changes
- [Bullet list from team reports]

## Test plan
- [x] [Automated tests that passed, with counts]
- [ ] [Manual verification step 1]
- [ ] [Manual verification step 2]
- [ ] [... one checkbox per user-facing behavior to verify]

## Acceptance Criteria
- [x] [AC 1]
- [x] [AC 2]
```

## Test Plan Guidelines

The test plan section is **checked during `/close-issue`** — unchecked items block the merge.

- Leave `[ ]` unchecked. The user checks these off after manually loading/testing the skill or plugin.
- Write items as concrete steps (action → expected result), e.g. "Load skill in Vibedata → confirm description appears correctly".
- Cover every user-visible change — not internal structure.

After creating the PR, link it to the Linear issue via `linear-server:update_issue`.

## Worktree Preservation

**Do NOT remove the worktree.** The user tests manually on it. Include the worktree path in the final status report.
