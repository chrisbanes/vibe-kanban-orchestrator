# vibe-kanban-orchestrator

Claude Code plugins that orchestrate coding workspaces from your issue board. Supports both [Vibe Kanban](https://vibekanban.com) and GitHub Projects as issue trackers, with Vibe Kanban workspaces handling the agent execution.

Run manually or set up on a cron to keep your backlog moving.

## Plugins

| Plugin | Skills | Issue Tracker |
|--------|--------|---------------|
| `vk-orchestrator` | `/vk-orchestrate`, `/plan-to-vk-project` | Vibe Kanban |
| `github-orchestrator` | `/gh-orchestrate`, `/plan-to-gh-project` | GitHub Projects |

## Prerequisites

- [Claude Code](https://claude.com/claude-code) CLI
- [Vibe Kanban](https://vibekanban.com) MCP server installed and configured (required for both — handles workspace execution)
- [`gh` CLI](https://cli.github.com/) installed and authenticated (GitHub Projects plugin only)

## Installation

Add the marketplace, then install the plugin(s) you want:

```
/plugin marketplace add chrisbanes/vibe-kanban-orchestrator
```

**For Vibe Kanban users:**

```
/plugin install vk-orchestrator@vibe-kanban-orchestrator
```

**For GitHub Projects users:**

```
/plugin install github-orchestrator@vibe-kanban-orchestrator
```

You can install both if you use both systems.

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
  "plan_directory": "docs/plans",
  "github": {
    "project_number": 1,
    "owner": "your-github-username"
  }
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
| `github.project_number` | — | GitHub Project board number (required for GitHub plugin) |
| `github.owner` | — | GitHub user or org that owns the project (required for GitHub plugin) |

> **Recommended setup for GitHub Projects:** Use a single project board per org or team that tracks issues across multiple repos. The orchestrator pulls work from one project board and matches each issue to the correct repo automatically. This avoids needing separate orchestrator runs per repo.

## Usage

### Orchestration

Run the orchestrator to check for work and start agents:

```
/vk-orchestrate        # Vibe Kanban
/gh-orchestrate        # GitHub Projects
```

Set up a recurring run:

```
/loop 10m /vk-orchestrate
/loop 10m /gh-orchestrate
```

Both orchestrators:

1. Check completed workspaces and mark done (VK updates issue status; GitHub relies on auto-close from PR merge)
2. Start code reviews for workspaces with open PRs
3. Flag stuck or failed workspaces
4. Pick up the next highest-priority unblocked task
5. Match it to a repo and start a new workspace
6. Report a summary of all actions taken

### Plan to Project

Create issues from a plan document:

```
/plan-to-vk-project    # Creates Vibe Kanban issues
/plan-to-gh-project    # Creates GitHub Issues with sub-issues
```

Both will find plan files in your configured `plan_directory`, ask you to pick one, set priority, and create issues. The GitHub variant creates a parent issue (epic) with sub-issues for each `### Task` section.

## License

Apache 2.0
