# vibe-kanban-orchestrator

A Claude Code plugin that orchestrates coding workspaces from your [Vibe Kanban](https://vibekanban.com) issue board.

Run manually or set up on a cron to keep your backlog moving.

> This repository is a Claude Code plugin marketplace — install individual plugins from it, don't install the repo directly.

## Skills

| Skill | Description |
|-------|-------------|
| `/vk-orchestrate` | Check for open tasks, manage workspace lifecycle, start new workspaces |
| `/plan-to-vk-project` | Create Vibe Kanban issues from a plan document |

## Prerequisites

- [Claude Code](https://claude.com/claude-code) CLI
- [Vibe Kanban](https://vibekanban.com) MCP server installed and configured

## Installation

Add the marketplace, then install the plugin:

```
/plugin marketplace add chrisbanes/vibe-kanban-orchestrator
/plugin install vk-orchestrator@vibe-kanban-orchestrator
```

## Configuration

Create `~/.vibe-kanban-orchestrate.json` to customize behavior:

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
  "plan_directory": "docs/plans"
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
| `plan_directory` | `"docs/plans"` | Directory to search for plan files |

## Usage

### Orchestration

Run the orchestrator to check for work and start agents:

```
/vk-orchestrate
```

Set up a recurring run:

```
/loop 10m /vk-orchestrate
```

The orchestrator:

1. Checks completed workspaces and marks issues as done
2. Starts code reviews for workspaces with open PRs
3. Flags stuck or failed workspaces
4. Picks up the next highest-priority unblocked task
5. Matches it to a repo and starts a new workspace
6. Reports a summary of all actions taken

### Plan to Project

Create issues from a plan document:

```
/plan-to-vk-project
```

This will find plan files in your configured `plan_directory`, ask you to pick one, set priority, and create issues. You can create a single issue for the whole plan or one issue per `### Task` section.

## License

Apache 2.0
