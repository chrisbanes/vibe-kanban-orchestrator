# vibe-kanban-orchestrator

A Claude Code plugin that orchestrates [Vibe Kanban](https://vibekanban.com) workspaces. It checks for open tasks, manages workspace lifecycle, and starts new workspaces for the next highest-priority task.

Run it manually or set it up on a cron to keep your backlog moving.

## Prerequisites

- [Claude Code](https://claude.com/claude-code) CLI
- [Vibe Kanban](https://vibekanban.com) MCP server installed and configured

## Installation

```
claude plugin add --from github:chrisbanes/vibe-kanban-orchestrator
```

## Configuration

Optionally create `~/.vibe-kanban-orchestrator.json` to customize behavior:

```json
{
  "max_concurrent_workspaces": 2,
  "default_branch": "main",
  "default_pre_prompt": "You are an autonomous coding agent working on a task from the backlog. Read the task description carefully, explore the codebase, implement changes with tests, follow existing conventions, and create a PR when done.",
  "project_overrides": {
    "my-project": {
      "pre_prompt": "Custom prompt for this project",
      "branch": "develop"
    }
  }
}
```

| Field | Default | Description |
|-------|---------|-------------|
| `max_concurrent_workspaces` | `2` | Maximum number of active workspaces at once |
| `default_branch` | `"main"` | Branch to use when starting new workspaces |
| `default_pre_prompt` | See above | System prompt prepended to each workspace task |
| `project_overrides` | `{}` | Per-project overrides for `pre_prompt` and `branch` |

## Usage

Invoke the `orchestrator` skill in Claude Code, or set up a recurring run:

```
/loop 10m /orchestrator
```

You can also use system cron to trigger it on a schedule.

## What it does

1. Checks completed workspaces and marks merged PRs as Done
2. Flags stuck or failed workspaces
3. Picks up the next highest-priority unblocked task
4. Matches it to a repo and starts a new workspace
5. Reports a summary of all actions taken

## License

Apache 2.0
