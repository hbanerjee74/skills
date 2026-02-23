---
name: implement-linear-issue
description: |
  Implements a Linear issue end-to-end, from planning through PR creation.
  Triggers on "implement <issue-id>", "work on <issue-id>", "working on <issue-id>", "build <issue-id>", "fix <issue-id>", or "/implement-issue".
  Also triggers when the user simply mentions a Linear issue identifier (e.g. "VD-123").
---

# Implement Linear Issue

You are a **coordinator**. Delegate all work to sub-agents via `Task`.

## Autonomy

Do not ask permission for non-destructive work. Only confirm with the user:
- The implementation plan (before execution)
- Scope changes discovered during implementation
- Final status before moving to review

## Scope Changes

When the user expands or changes scope during the conversation, update the Linear issue immediately — add new ACs, update the description. If work streams are in flight, assess whether changes invalidate their work before continuing.

## Setup (do these steps exactly)

1. Fetch the issue via `linear-server:get_issue`. Get: ID, title, description, requirements, acceptance criteria, estimate, branchName, **status**.
2. **Check for child issues:** Fetch children via `linear-server:list_issues` with `parentId` set to this issue's ID. If children exist:
   - Present the child issues (ID, title, status, estimate) to the user.
   - Ask: implement all children together in a single worktree, or just the parent?
   - If the user chooses **all children together**:
     - Use the **parent issue's branchName** for the worktree.
     - Collect requirements and ACs from all child issues (fetch each via `linear-server:get_issue`).
     - Apply status guards (step 3) to the parent and each included child — assign to me, move to In Progress.
     - During planning, present a unified plan covering all children. Map each work stream to its source child issue.
     - During implementation, check off ACs on each child's Linear issue as they're completed.
     - During completion, create a **single PR** with `Fixes <child-id>` on separate lines for each child (see `git-and-pr.md`). Move all children to Review.
   - If the user chooses **parent only**, proceed normally with just the parent issue.
3. **Guard on status:**
   - **Done / Cancelled / Duplicate** → Stop and tell the user. Do not proceed.
   - **Todo** → Assign to me + move to In Progress via `linear-server:update_issue` (`assignee: "me"`, `state: "In Progress"`).
   - **In Progress** → Already active. Skip status change (assign to me if unassigned). Resume work.
   - **In Review** → Move back to In Progress via `linear-server:update_issue`. Resume work — likely addressing review feedback or continuing the pipeline.
4. **Create a git worktree** at `../worktrees/<branchName>` using the `branchName` from the issue (or parent's branchName if implementing children together). Reuse if it already exists. All subsequent sub-agents work in this worktree path, NOT the main repo.

## Objectives

Given the issue (or set of child issues), deliver a working implementation that satisfies all acceptance criteria and is ready for human review. How you get there depends on the issue's complexity and constraints. Track these outcomes:

- Issue(s) assigned and worktree ready
- Plan approved by user
- All ACs implemented and checked off on Linear (each child issue separately if multi-child)
- Markdown linted and content reviewed
- Documentation updated
- Marketplace synced
- PR created and linked

**Plan approval checkpoint:**
1. Present the plan to the user. Iterate through feedback until the user says to proceed.
2. When the user approves, **post the approved plan to Linear** as a comment on the issue (via `linear-server:create_comment`) — then start coding. Linear always gets the final agreed plan, never a draft.

**During implementation:**
- Each agent checks off its ACs on Linear after completing them via `linear-server:update_issue`.
- When the user changes scope or rejects an approach mid-implementation, update the Linear issue description/ACs immediately.

**After every change-making turn:**
- Flag any gaps between implemented changes and the issue's requirements/ACs. If the user agrees to adjustments, update the Linear issue.

## Post-Implementation Quality Gates

After all ACs are implemented, achieve every gate below. Parallelize independent gates.

### Dependency Graph

```
markdown linted ──→ content reviewed ──┐
                                       ├──→ marketplace updated → PR
docs updated (independent) ────────────┘
```

Copy this checklist and check off gates as they pass:

```
Quality Gates:
- [ ] Markdown linted
- [ ] Content reviewed
- [ ] Docs updated
- [ ] Marketplace updated
```

### Markdown linted

1. Read [quality-checklist.md](references/quality-checklist.md).
2. Run `git diff main --name-only` in the worktree to identify changed files.
3. For every changed `SKILL.md`: go through each item in the **SKILL.md authoring** section of the checklist. Check it off or note the violation.
4. For every changed `plugin.json` or `marketplace.json`: go through each item in the **marketplace.json schema** section. Check it off or note the violation.
5. Run `claude plugin validate .` from the repo root — fix any errors it reports.
6. Fix all violations, then re-verify the affected items before proceeding.

### Content reviewed

Spawn `feature-dev:code-reviewer`. See [review-flow.md](references/review-flow.md). Reviewer checks content accuracy, clarity, and completeness against the issue's ACs. Max 2 review-fix cycles — fix high/medium issues, note low-severity if not straightforward.

### Docs updated

Keep project docs in sync. Check `CLAUDE.md` and any `README.md` in changed directories. Update if the changes affect documented conventions, structure, or usage. Commit doc updates separately. No dependency on review — can run in parallel.

### Marketplace updated

Run the `/update-marketplace` skill (follow `.claude/skills/update-marketplace/SKILL.md`). This syncs `.claude-plugin/marketplace.json` with the current state of `plugins/` and `skills/`. Commit the result. This is the last gate before raising the PR.

## Completion

Enter when all pipeline phases pass.

1. Verify all ACs are checked on Linear — for every issue being implemented (parent + children if multi-child). If any missed, spawn a fix agent and re-verify.
2. Create PR and link to issue(s). See [git-and-pr.md](references/git-and-pr.md). For multi-child: single PR with `Fixes <child-id>` on separate lines for each child issue.
3. **Generate implementation notes** from the code (primary) and conversation context (supplemental):
   - Run `git log --oneline main..HEAD` and `git diff main --stat` to get what actually shipped.
   - Synthesize into a structured comment: what was implemented, files changed, key decisions made during implementation.
   - Show the notes to the user for review/edits.
   - When the user approves, post to Linear as a comment on each issue (via `linear-server:create_comment`).
4. Move issue(s) to Review via `linear-server:update_issue`. For multi-child: move all children to Review.
5. Report to user with: PR link, worktree path, and manual validation steps from the PR test plan.
6. **Do NOT remove the worktree** — user tests manually on it.

## Sub-agent Delegation

Follows the project's Delegation Policy in CLAUDE.md (model tiers, sub-agent rules, output caps).

Skill-specific rules:
- Always provide the **worktree path** (not main repo path)
- Sub-agents can spawn their own sub-agents for parallelism

## Error Recovery

| Situation | Action |
|---|---|
| Sub-agent fails | Max 2 retries, then escalate to user |
| Worktree exists on wrong branch | Remove and recreate |
| Linear API fails | Retry once, then continue and note for user |
| Scope changes mid-execution | Reassess in-flight work, re-plan if needed |
| ACs remain unmet after fixes | Keep In Progress, report to user |
