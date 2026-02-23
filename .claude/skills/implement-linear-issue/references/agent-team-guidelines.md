# Agent Team Guidelines

## Team Leads

Each work stream gets a team lead (spawned via `Task`). Provide: worktree path, issue ID, the ACs this stream owns, task list, and dependencies.

Team leads coordinate within their stream — spawning sub-agents for parallel tasks, not writing code themselves.

### Rules
- Commit + push before reporting (conventional commit format)
- Check off your ACs on Linear after tests pass
- Report back: what completed, tests updated/added/removed, ACs addressed, blockers
- Do NOT write to the Implementation Updates section (coordinator-only)

## Failure Handling

Max 2 retries per team before escalating to user. Pause dependent streams if a blocking failure occurs.
