# Code Review Stage Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add an automated code review step to the orchestrator skill so that workspaces with open PRs get reviewed by a separate agent session before human merge.

**Architecture:** New Step 3 inserted into SKILL.md between "Check Completed Work" and the current "Check Workspace Health". Config gains a nested `review` block. `default_pre_prompt` renamed to `prompt`. `project_overrides` removed entirely. README updated to match.

**Tech Stack:** Markdown (SKILL.md skill definition), JSON (config schema)

**Design doc:** `docs/plans/2026-03-14-code-review-stage-design.md`

---

### Task 1: Update Step 1 (Load Config) in SKILL.md

**Files:**
- Modify: `skills/orchestrator/SKILL.md:10-22`

**Step 1: Edit the config loading section**

Replace the current Step 1 content with updated config that:
- Renames `default_pre_prompt` to `prompt`
- Removes `project_overrides`
- Adds `review` block with `enabled`, `executor`, `variant`, `prompt`

New content for Step 1:

```markdown
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
```

**Step 2: Commit**

```bash
git add skills/orchestrator/SKILL.md
git commit -m "feat: update config schema — rename prompt, add review block, remove project_overrides"
```

---

### Task 2: Insert new Step 3 (Check for Reviewable Work) in SKILL.md

**Files:**
- Modify: `skills/orchestrator/SKILL.md` (insert after Step 2, before current Step 3)

**Step 1: Insert the new step after Step 2**

Add the following section between the existing Step 2 and Step 3:

```markdown
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
```

**Step 2: Renumber existing Steps 3-9 to Steps 4-10**

Update all step headers:
- Step 3 → Step 4
- Step 4 → Step 5
- Step 5 → Step 6
- Step 6 → Step 7
- Step 7 → Step 8
- Step 8 → Step 9
- Step 9 → Step 10

Also update any cross-references within the steps:
- Step 4 (old Step 3): references "Step 2" — no change needed
- Step 5 (old Step 4): "Skip to Step 9 (Report)" → "Skip to Step 10 (Report)"
- Step 7 (old Step 6): "skip to Step 9" → "skip to Step 10"
- Step 8 (old Step 7): "skip to Step 9" → "skip to Step 10"

**Step 3: Commit**

```bash
git add skills/orchestrator/SKILL.md
git commit -m "feat: add Step 3 — Check for Reviewable Work, renumber existing steps"
```

---

### Task 3: Update Step 9 (Start Workspace) to use renamed config fields

**Files:**
- Modify: `skills/orchestrator/SKILL.md` (the step formerly known as Step 8, now Step 9)

**Step 1: Update the Start Workspace step**

Replace references to old config fields:
- `project_overrides[project_name].pre_prompt` → remove (no more project overrides)
- `default_pre_prompt` → `prompt`
- `project_overrides[project_name].branch` → remove
- `default_branch` remains as-is

New content for the prompt/branch determination:

```markdown
## Step 9: Start Workspace

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
```

**Step 2: Commit**

```bash
git add skills/orchestrator/SKILL.md
git commit -m "feat: update Start Workspace step to use renamed config fields"
```

---

### Task 4: Update the Report Summary (Step 10)

**Files:**
- Modify: `skills/orchestrator/SKILL.md` (the step formerly known as Step 9, now Step 10)

**Step 1: Add Reviews section to the report**

Update the report template to include a "Reviews" section after "Completed":

```markdown
## Step 10: Report Summary

Output a summary with these sections:

\`\`\`
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
\`\`\`
```

**Step 2: Commit**

```bash
git add skills/orchestrator/SKILL.md
git commit -m "feat: add Reviews section to orchestration report"
```

---

### Task 5: Update README.md

**Files:**
- Modify: `README.md`

**Step 1: Update the Configuration section**

Replace the config example and table to reflect:
- `default_pre_prompt` → `prompt`
- Remove `project_overrides`
- Add `review` block

New config example:

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
  }
}
```

New table:

| Field | Default | Description |
|-------|---------|-------------|
| `max_concurrent_workspaces` | `2` | Maximum number of active workspaces at once |
| `default_branch` | `"main"` | Branch to use when starting new workspaces |
| `prompt` | See above | System prompt prepended to each workspace task |
| `review.enabled` | `true` | Whether to auto-review PRs before merge |
| `review.executor` | server default | Executor for the review session |
| `review.variant` | server default | Variant (model) for the review session |
| `review.prompt` | See above | Prompt given to the review agent |

**Step 2: Update the "What it does" section**

Add the review step to the numbered list:

```markdown
## What it does

1. Checks completed workspaces and marks merged PRs as Done
2. Starts code reviews for workspaces with open PRs
3. Flags stuck or failed workspaces
4. Picks up the next highest-priority unblocked task
5. Matches it to a repo and starts a new workspace
6. Reports a summary of all actions taken
```

**Step 3: Fix the config filename**

The README references `~/.vibe-kanban-orchestrator.json` (with an `r` at the end) but the SKILL.md uses `~/.vibe-kanban-orchestrate.json` (no `r`). Fix the README to match SKILL.md: `~/.vibe-kanban-orchestrate.json`.

**Step 4: Commit**

```bash
git add README.md
git commit -m "docs: update README with review config and updated field names"
```
