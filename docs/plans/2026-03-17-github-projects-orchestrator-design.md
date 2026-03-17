# GitHub Projects Orchestrator

## Goal

Add a new `github-orchestrator` plugin that provides GitHub Projects equivalents of the existing Vibe Kanban orchestration and plan-to-board skills. Users can install either or both plugin sets.

## Motivation

Not all users use Vibe Kanban as their issue tracker. Many teams already use GitHub Projects for task management. This plugin lets them use the same orchestration workflow with GitHub Projects as the source of truth, while still using Vibe Kanban workspaces for agent execution.

## Architecture

### Plugin Structure

Rename existing skills and consolidate into two plugins:

```
plugins/
  vk-orchestrator/
    .claude-plugin/
      plugin.json
    skills/
      vk-orchestrate/SKILL.md
      plan-to-vk-project/SKILL.md
  github-orchestrator/
    .claude-plugin/
      plugin.json
    skills/
      gh-orchestrate/SKILL.md
      plan-to-gh-project/SKILL.md
```

The marketplace.json lists both plugins. Users install one or both:

```
/plugin install vk-orchestrator@vibe-kanban-orchestrator
/plugin install github-orchestrator@vibe-kanban-orchestrator
```

### Skill Names

| Old Name | New Name (VK) | New Name (GitHub) |
|----------|--------------|-------------------|
| orchestrate | vk-orchestrate | gh-orchestrate |
| plan-to-board | plan-to-vk-project | plan-to-gh-project |

### Config

Both variants share `~/.vibe-kanban-orchestrate.json`. The GitHub variant adds a `github` section:

```json
{
  "max_concurrent_workspaces": 2,
  "default_branch": "main",
  "prompt": "...",
  "review": {
    "enabled": true,
    "prompt": "..."
  },
  "github": {
    "project_number": 1,
    "owner": "chrisbanes"
  }
}
```

- `github.project_number` — the GitHub Project board number
- `github.owner` — the GitHub user or org that owns the project

### Status Model

GitHub Projects uses three statuses (simpler than Vibe Kanban's five):

| Status | Meaning |
|--------|---------|
| Todo | Eligible for pickup |
| In Progress | Workspace running |
| Done | PR merged (handled automatically by GitHub) |

Review still happens (agent starts a review session when a PR appears), but does not surface as a separate board status. The GitHub PR review process is the human review gate.

### Issue-to-Workspace Linking

Workspaces are named `#N Issue title` (e.g., `#42 Fix login bug`). The `#N` prefix is parsed to reliably find the corresponding GitHub issue. Title is included for readability and as a fallback.

### Execution Layer

Both variants use Vibe Kanban workspaces for execution (git worktrees, agent sessions, prompt delivery). The difference is only where issues are read from and where status is synced to.

## `gh-orchestrate` Skill Flow

### Step 1: Load Config

Read `~/.vibe-kanban-orchestrate.json` for:
- `github.project_number` and `github.owner` (required)
- `max_concurrent_workspaces` (default: 2)
- `default_branch` (default: "main")
- `prompt` (default agent prompt)
- `review` settings

### Step 2: Check Completed Work

1. Call `list_workspaces(archived: false)` to get all active workspaces.
2. For each workspace, parse `#N` from the workspace name.
3. Use `gh issue view N --repo REPO --json state` to check if the issue is closed.
4. If closed (PR merged and GitHub auto-closed it), archive the workspace.
5. Report: "Archived workspace for #N (issue closed)"

### Step 3: Check for Reviewable Work

1. If `review.enabled` is false, skip.
2. For each active workspace, parse `#N` and check for an open PR linked to the issue.
3. Use `gh pr list --search "closes #N"` or check issue timeline for linked PRs.
4. If an open PR exists and no review session has been started yet:
   - Call `create_session(workspace_id)` with review executor/variant if configured.
   - Call `run_session_prompt(session_id, review.prompt)`.
   - Report: "Started review for #N: <title>"

### Step 4: Check Workspace Health

1. Check workspace `updated_at` — if not updated in over 2 hours, flag as potentially stuck.
2. Check if linked PR was closed without merging — flag as failed.
3. Report warnings. Do NOT automatically take action.

### Step 5: Check Concurrency

1. Count active (non-archived) workspaces.
2. If count >= `max_concurrent_workspaces`:
   - Report: "Max concurrency reached (<count>/<max>). Not picking up new work."
   - Skip to Step 9.

### Step 6: Gather Eligible Work

1. Use `gh project item-list <project_number> --owner <owner>` to list project items.
2. Filter to items with status "Todo" that are real issues (not drafts).
3. Collect issue numbers, titles, and repos.

### Step 7: Filter by Dependencies

1. For each candidate issue, use `gh api` GraphQL to check for sub-issue relationships.
2. If the issue is a sub-issue of a parent that has other incomplete sub-issues that block it, skip.
3. Sort eligible issues by priority labels (e.g., `priority:urgent` > `priority:high` > `priority:medium` > `priority:low`), then by creation date ascending.

If no eligible issues, report "No eligible work found" and skip to Step 9.

### Step 8: Start Workspace

1. Find the matching Vibe Kanban repo by name (from `list_repos`).
2. If no match, skip to next eligible issue.
3. Call `start_workspace` with:
   - `name`: `#N <issue title>`
   - `executor`: `"CLAUDE_CODE"`
   - `repositories`: `[{ repo_id, branch: default_branch }]`
   - `prompt`: `<prompt>\n\nTask:\n<issue title>\n<issue description>`
4. Move the GitHub Project item to "In Progress" via GraphQL `updateProjectV2ItemFieldValue` mutation.
5. Report: "Started workspace for #N: <title> (repo: <repo>, branch: <branch>)"

### Step 9: Report Summary

```
## Orchestration Report

### Completed
- <workspaces archived, or "None">

### Reviews
- <reviews started, or "None">

### Health Warnings
- <stuck or failed workspaces, or "None">

### Started
- <new workspace started, or "No new work picked up">

### Skipped
- <issues skipped with reasons, or "None">

### Status
- Active workspaces: <count>/<max>
```

## `plan-to-gh-project` Skill Flow

### Step 1: Load Config

Read `~/.vibe-kanban-orchestrate.json` for:
- `github.project_number` and `github.owner` (required)
- `plan_directory` (default: `docs/plans`)

### Step 2: Find Plans

Glob for `{plan_directory}/*.md`. If none found, report and stop.

### Step 3: Select Plan

If multiple plans, list them and ask the user to pick one. Read the selected plan.

### Step 4: Extract Title

Use the first `# ` heading as the parent issue title. Fall back to filename.

### Step 5: Determine Repo

1. If running inside a git repo, use that repo (detect via `gh repo view`).
2. If not in a repo, list repos with `gh repo list {owner}` and ask the user to pick one.

### Step 6: Set Priority

Scan plan content for priority keywords. Ask user to confirm. Map to GitHub labels (`priority:urgent`, `priority:high`, `priority:medium`, `priority:low`).

### Step 7: Create Parent Issue

1. `gh issue create --title "<title>" --body "<plan header content>" --label "<priority label>"` in the determined repo.
2. Add to GitHub Project via `gh project item-add <project_number> --owner <owner> --url <issue_url>`.

### Step 8: Create Sub-Issues

1. Parse plan for `### Task` sections.
2. For each task section:
   - `gh issue create --title "<task heading>" --body "<task content>" --label "<priority label>"`
   - Link as sub-issue to parent via `gh api` GraphQL.
   - Add to GitHub Project.
3. Set all items to "Todo" status.

### Step 9: Report

```
## Plan to GitHub Project

- **Issues created:** <count> (1 parent + N sub-issues)
- **Repo:** <repo name>
- **Project:** <project name>
- **Priority:** <priority>

### Issues
- #<number> — <parent title> (parent)
  - #<number> — <task title>
  - #<number> — <task title>
  - ...

Run `/gh-orchestrate` to pick up these issues, or wait for the next scheduled run.
```

## What Changes

- Rename `plugins/orchestrator/` → `plugins/vk-orchestrator/`
- Rename `plugins/plan-to-board/` → inside `plugins/vk-orchestrator/`
- Merge existing VK skills into single `vk-orchestrator` plugin
- Rename skill `orchestrate` → `vk-orchestrate`
- Rename skill `plan-to-board` → `plan-to-vk-project`
- Create new `plugins/github-orchestrator/` with `gh-orchestrate` and `plan-to-gh-project` skills
- Update `marketplace.json` to list `vk-orchestrator` and `github-orchestrator`

## What Stays the Same

- All Vibe Kanban skill logic (just renamed)
- Config file path (`~/.vibe-kanban-orchestrate.json`)
- Vibe Kanban workspace execution layer (used by both variants)

## Dependencies

- `gh` CLI must be installed and authenticated
- GitHub Project must exist with Todo/In Progress/Done statuses
- Vibe Kanban MCP server must be running (for workspace management)
- Sub-issue creation requires `gh api` GraphQL (no native CLI support yet)
