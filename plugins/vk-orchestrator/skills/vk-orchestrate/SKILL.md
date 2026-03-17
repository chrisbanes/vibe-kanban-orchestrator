---
name: vk-orchestrate
description: Check Vibe Kanban for open tasks, manage workspace lifecycle, start new workspaces for the next highest-priority task. Run manually or via cron.
---

# Agent Orchestration

You are an orchestrator. Follow these steps exactly.

## Step 1: Load Config

Read `~/.vibe-kanban-orchestrate.json` to get:
- `max_concurrent_workspaces` (default: 2)
- `default_branch` (default: "main")
- `prompt`
- `review` (nested object with `enabled`, `executor`, `variant`, `prompt`)

If the file doesn't exist or is invalid, use these defaults:
- max_concurrent_workspaces: 2
- default_branch: "main"
- prompt: "You are an autonomous coding agent working on a task from the backlog. Read the task description carefully, explore the codebase, implement changes with tests, follow existing conventions, and create a PR when done."
- review:
  - enabled: true
  - executor: (not set — uses server default)
  - variant: (not set — uses server default)
  - prompt: "You are a code reviewer. Review the open PR in this workspace. Examine all changes for bugs, code quality issues, missing tests, and deviations from the issue requirements. Fix any issues you find and push your changes."

## Step 2: Check Completed Work

1. Call `list_workspaces(archived: false)` to get all active workspaces.
2. For each workspace that has a linked issue:
   - Call `get_issue(issue_id)` to check the issue state.
   - If `pull_requests` contains a PR with a merged status **and** the issue status is "Human Review", call `update_issue(issue_id, status: "Done")`.
   - Report: "Marked <issue simple_id> as Done (PR merged)".

## Step 3: Check for Reviewable Work

1. If `review.enabled` is `false`, skip this step.
2. Iterate over active workspaces from Step 2.
3. For each workspace with a linked issue:
   - Call `get_issue(issue_id)` to check the issue state.
   - If `latest_pr_status` shows an open PR **and** the issue status is "In Progress":
     - Call `create_session(workspace_id)` — pass `executor` and/or `variant` from `review` config only if set.
     - Call `run_session_prompt(session_id, <review.prompt>)`.
     - Call `update_issue(issue_id, status: "In Review")`.
     - Report: "Started review for <issue simple_id>: <issue title>"

## Step 4: Check for Human Review

1. Iterate over active workspaces from Step 2.
2. For each workspace with a linked issue:
   - Call `get_issue(issue_id)` to check the issue state.
   - If the issue status is "In Review":
     - Call `list_sessions(workspace_id)` to check review session status.
     - If the most recent session has completed (is no longer running), call `update_issue(issue_id, status: "Human Review")`.
     - Report: "Moved <issue simple_id> to Human Review (automated review complete)"

## Step 5: Check Workspace Health

Review each non-archived workspace from Step 2:

1. Check workspace `updated_at` — if it hasn't been updated in over 2 hours, flag it as potentially stuck.
2. Check linked issue's `latest_pr_status` — if the PR was closed (not merged), flag the workspace as failed.
3. Report any stuck or failed workspaces so the user is aware. Do NOT automatically take action on these — just report them.

## Step 6: Check Concurrency

1. Count non-archived workspaces (from Step 2 results), excluding any whose linked issue status is "Human Review".
2. If count >= `max_concurrent_workspaces` from config:
   - Report: "Max concurrency reached (<count>/<max>). Not picking up new work."
   - Skip to Step 11 (Report).

## Step 7: Gather Eligible Work

1. Call `list_organizations` to get all orgs.
2. For each org, call `list_projects(organization_id)`.
3. For each project, call `list_issues(project_id, status: "To do")`.
4. Collect all "To do" issues, keeping track of which project each belongs to.

## Step 8: Filter by Dependencies

For each candidate issue:

1. Call `get_issue(issue_id)` to get full details including `relationships` and `sub_issues`.
2. Check `relationships` — if any relationship has `relationship_type: "blocking"` where the blocking issue's status is NOT "Done", skip this issue.
3. Issues with no blocking dependencies are eligible.

Sort eligible issues:
- By priority: urgent (1) > high (2) > medium (3) > low (4) > null (5)
- Within same priority: by `created_at` ascending (oldest first)

If no eligible issues exist, report "No eligible work found" and skip to Step 11.

## Step 9: Match Repo

1. Call `list_repos` to get all available repos.
2. Take the selected issue's project name and find a repo with a matching name (case-insensitive).
3. If no match: skip this issue, try the next eligible issue. If all exhausted, report "No repo match found for any eligible issue" and skip to Step 11.

## Step 10: Start Workspace

1. Determine the prompt: use `prompt` from config.
2. Determine the branch: use `default_branch` from config.
3. Call `start_workspace` with:
   - `name`: The issue title
   - `executor`: `"CLAUDE_CODE"`
   - `repositories`: `[{ repo_id: <matched_repo_id>, branch: <branch> }]`
   - `issue_id`: The issue's ID
   - `prompt`: `<prompt>\n\nTask:\n<issue title>\n<issue description>`
4. Call `update_issue(issue_id, status: "In Progress")`.
5. Report: "Started workspace for <issue simple_id>: <issue title> (repo: <repo name>, branch: <branch>)"

## Step 11: Report Summary

Output a summary with these sections:

```
## Orchestration Report

### Completed
- <issues marked as Done, or "None">

### Reviews
- <reviews started, or "None">

### Awaiting Human Review
- <issues moved to Human Review, or "None">

### Health Warnings
- <stuck or failed workspaces, or "None">

### Started
- <new workspace started, or "No new work picked up">

### Skipped
- <issues skipped with reasons, or "None">

### Status
- Active workspaces: <count>/<max>
```
