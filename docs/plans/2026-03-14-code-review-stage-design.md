# Code Review Stage Design

## Summary

Add an automated code review step to the orchestrator skill. When a workspace has an open PR and its issue is "In Progress", the orchestrator starts a new review session in that workspace. The review agent fixes issues directly and pushes changes. After review, the issue moves to "Reviewed" for human merge.

## Config Changes

### Rename

`default_pre_prompt` renamed to `prompt`.

### New `review` block

The global config (`~/.vibe-kanban-orchestrate.json`) gains a nested `review` object:

```json
{
  "max_concurrent_workspaces": 2,
  "default_branch": "main",
  "prompt": "You are an autonomous coding agent...",
  "review": {
    "enabled": true,
    "executor": "CLAUDE_CODE",
    "variant": "sonnet",
    "prompt": "You are a code reviewer. Review the open PR in this workspace. Examine all changes for bugs, code quality issues, missing tests, and deviations from the issue requirements. Fix any issues you find and push your changes."
  }
}
```

All fields in `review` are optional:

- `enabled` — defaults to `true`
- `executor` — not set by default, uses server default
- `variant` — not set by default, uses server default
- `prompt` — defaults to the built-in review prompt above

## New Step 3: Check for Reviewable Work

Inserted after Step 2 (Check Completed Work). Existing Steps 3-9 shift to Steps 4-10.

### Logic

1. If `review.enabled` is `false`, skip this step.
2. Iterate over active workspaces from Step 2.
3. For each workspace with a linked issue:
   - Call `get_issue(issue_id)` to check the issue state.
   - If `latest_pr_status` shows an open PR **and** the issue status is "In Progress":
     - Call `create_session(workspace_id, executor, variant)` — only passing `executor`/`variant` if set in config.
     - Call `run_session_prompt(session_id, review.prompt)`.
     - Call `update_issue(issue_id, status: "In Review")`.
     - Report: "Started review for \<issue simple_id\>: \<issue title\>"

### Guards

- Only issues with status "In Progress" get reviewed, preventing re-triggers on "In Review" or "Reviewed" issues.
- Reviews reuse existing workspaces and do not count against `max_concurrent_workspaces`.

## Changes to Existing Steps

### Step 2: Check Completed Work

No logic change. Issues in "In Review" or "Reviewed" status with merged PRs are already handled — the existing merged-PR check applies regardless of status.

### Step 4 (was Step 3): Check Workspace Health

No change. Existing logic already checks `latest_pr_status` regardless of issue status, so "In Review" workspaces with closed (non-merged) PRs are flagged as failed.

## Report Changes (Step 10)

Add a "Reviews" section after "Completed":

```
## Orchestration Report

### Completed
- <issues marked as Done, or "None">

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

## Issue Status Flow

```
To do → In Progress → In Review → Reviewed → (human merges PR) → Done
```
