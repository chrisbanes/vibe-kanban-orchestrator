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

Optionally create `~/.vibe-kanban-orchestrate.json` to customize behavior:

```json
{
  "max_concurrent_workspaces": 2,
  "default_branch": "main",
  "prompt": "You are an autonomous coding agent working on a task from the backlog. Read the task description carefully, explore the codebase, implement changes with tests, follow existing conventions, and create a PR when done.",
  "review": {
    "enabled": true,
    "executor": "CLAUDE_CODE",
    "variant": "sonnet",
    "prompt": "You are a code reviewer. Review the open PR in this workspace. Examine all changes for bugs, code quality issues, missing tests, and deviations from the issue requirements. Fix any issues you find and push your changes."
  },
  "plan_directory": "docs/plans/"
}
```

| Field | Default | Description |
|-------|---------|-------------|
| `max_concurrent_workspaces` | `2` | Maximum number of active workspaces at once |
| `default_branch` | `"main"` | Branch to use when starting new workspaces |
| `prompt` | See above | System prompt prepended to each workspace task |
| `review.enabled` | `true` | Whether to auto-review PRs before merge |
| `review.executor` | server default | Executor for the review session |
| `review.variant` | server default | Variant (model) for the review session |
| `review.prompt` | See above | Prompt given to the review agent |
| `plan_directory` | `"docs/plans/"` | Directory to search for plan files |

## Usage

Invoke the `orchestrator` skill in Claude Code, or set up a recurring run:

```
/loop 10m /orchestrator
```

You can also use system cron to trigger it on a schedule.

## What it does

1. Checks completed workspaces and marks merged PRs as Done
2. Starts code reviews for workspaces with open PRs
3. Flags stuck or failed workspaces
4. Picks up the next highest-priority unblocked task
5. Matches it to a repo and starts a new workspace
6. Reports a summary of all actions taken

## Plan to Board

Use `/plan-to-board` to create a Vibe Kanban issue from a plan document. This bridges plan authoring (e.g., from a brainstorming/planning session) with orchestrated execution.

```
/plan-to-board
```

It will:

1. Find plan files in your configured `plan_directory`
2. Ask you to pick one (if multiple)
3. Ask which project and priority
4. Create an issue with the full plan as the description
5. Optionally update your orchestrator prompt for plan-based execution

## License

Apache 2.0
