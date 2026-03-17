---
name: gh-orchestrate
description: Check GitHub Projects for open tasks, manage workspace lifecycle, start new workspaces for the next highest-priority task. Run manually or via cron.
---

# Agent Orchestration (GitHub Projects)

You are an orchestrator. Follow these steps exactly.

## Step 1: Load Config

Read `~/.vibe-kanban-orchestrate.json` to get:
- `github.project_number` (required)
- `github.owner` (required)
- `max_concurrent_workspaces` (default: 2)
- `default_branch` (default: "main")
- `prompt` (default: "You are an autonomous coding agent working on a task from the backlog. Read the task description carefully, explore the codebase, implement changes with tests, follow existing conventions, and create a PR when done.")
- `review` (nested object with `enabled`, `executor`, `variant`, `prompt`)
  - review defaults:
    - enabled: true
    - executor: (not set — uses server default)
    - variant: (not set — uses server default)
    - prompt: "You are a code reviewer. Review the open PR in this workspace. Examine all changes for bugs, code quality issues, missing tests, and deviations from the issue requirements. Fix any issues you find and push your changes."

If the file doesn't exist, is invalid, or is missing `github.project_number` or `github.owner`, report the error and stop. These fields are required.

## Step 2: Check Completed Work

1. Call `list_workspaces(archived: false)` to get all active workspaces.
2. For each workspace, parse `#N` from the workspace name (expected format: `#N Issue title`).
3. For each parsed issue number, determine the repo from the workspace's repository info.
4. Run `gh issue view N --repo <owner>/<repo> --json state,stateReason` to check if the issue is closed.
5. If the issue state is "CLOSED", archive the workspace by calling `update_workspace(workspace_id, archived: true)`.
6. Report: "Archived workspace for #N (issue closed)"

## Step 3: Check for Reviewable Work

1. If `review.enabled` is `false`, skip this step.
2. For each active (non-archived) workspace, parse `#N` from the workspace name.
3. Determine the repo from the workspace's repository info.
4. Run `gh pr list --repo <owner>/<repo> --search "head:<workspace-branch>" --json number,state` to find PRs from the workspace branch.
5. If an open PR exists:
   - Call `list_sessions(workspace_id)` to check if a review session already exists.
   - If no review session has been run yet:
     - Call `create_session(workspace_id)` — pass `executor` and/or `variant` from `review` config only if set.
     - Call `run_session_prompt(session_id, <review.prompt>)`.
     - Report: "Started review for #N: <title>"

## Step 4: Check Workspace Health

Review each non-archived workspace:

1. Check workspace `updated_at` — if it hasn't been updated in over 2 hours, flag it as potentially stuck.
2. For each workspace, parse `#N` and determine the repo from workspace repository info.
3. Check if a linked PR was closed without merging:
   ```
   gh pr list --repo <owner>/<repo> --search "head:<workspace-branch>" --state closed --json mergedAt
   ```
   If the PR is closed and `mergedAt` is null, flag the workspace as failed.
4. Report any stuck or failed workspaces so the user is aware. Do NOT automatically take action on these — just report them.

## Step 5: Check Concurrency

1. Count active (non-archived) workspaces.
2. If count >= `max_concurrent_workspaces` from config:
   - Report: "Max concurrency reached (<count>/<max>). Not picking up new work."
   - Skip to Step 9 (Report).

## Step 6: Gather Eligible Work

1. Run the following to list all project items:
   ```
   gh project item-list <project_number> --owner <owner> --format json
   ```
2. Filter to items where:
   - The Status field equals "Todo"
   - The item type is "Issue" (not "DraftIssue" or "PullRequest")
3. For each eligible item, extract: issue number, title, and repository (owner/repo).

## Step 7: Filter by Dependencies

For each candidate issue:

1. Check for sub-issues using this GraphQL query:
   ```
   gh api graphql -f query='
     query($owner: String!, $repo: String!, $number: Int!) {
       repository(owner: $owner, name: $repo) {
         issue(number: $number) {
           subIssues(first: 50) {
             nodes {
               number
               state
               title
             }
           }
         }
       }
     }
   ' -f owner="<owner>" -f repo="<repo>" -F number=<N>
   ```
2. If the issue has sub-issues and any sub-issue has state "OPEN", skip the parent issue (work on sub-issues first).
3. Issues with no sub-issues, or whose sub-issues are all "CLOSED", are eligible.

Sort eligible issues:
- By priority labels on the issue: `priority:urgent` (1) > `priority:high` (2) > `priority:medium` (3) > `priority:low` (4) > no priority label (5)
- To check labels, run: `gh issue view <N> --repo <owner>/<repo> --json labels`
- Within same priority: by creation date ascending (oldest first)

If no eligible issues exist, report "No eligible work found" and skip to Step 9.

## Step 8: Start Workspace

1. Call `list_repos` to get all available Vibe Kanban repos.
2. Match the issue's repository name to a VK repo name (case-insensitive).
3. If no match: skip this issue, try the next eligible issue. If all exhausted, report "No repo match found for any eligible issue" and skip to Step 9.
4. Get the issue description:
   ```
   gh issue view <N> --repo <owner>/<repo> --json title,body
   ```
5. Call `start_workspace` with:
   - `name`: `#N <issue title>`
   - `executor`: `"CLAUDE_CODE"`
   - `repositories`: `[{ repo_id: <matched_repo_id>, branch: <default_branch> }]`
   - `prompt`: `<prompt>\n\nTask:\n<issue title>\n<issue body>`
6. Move the GitHub Project item to "In Progress":
   - Get the project's node ID, status field ID, and "In Progress" option ID. First try as an organization, and if that fails, retry as a user:
     ```
     gh api graphql -f query='
       query($owner: String!, $number: Int!) {
         organization(login: $owner) {
           projectV2(number: $number) {
             id
             field(name: "Status") {
               ... on ProjectV2SingleSelectField {
                 id
                 options {
                   id
                   name
                 }
               }
             }
           }
         }
       }
     ' -f owner="<owner>" -F number=<project_number>
     ```
     If the above query returns an error (e.g., "Could not resolve to an Organization"), retry with `user` instead:
     ```
     gh api graphql -f query='
       query($owner: String!, $number: Int!) {
         user(login: $owner) {
           projectV2(number: $number) {
             id
             field(name: "Status") {
               ... on ProjectV2SingleSelectField {
                 id
                 options {
                   id
                   name
                 }
               }
             }
           }
         }
       }
     ' -f owner="<owner>" -F number=<project_number>
     ```
   - Find the option ID where `name` equals "In Progress".
   - Get the item ID from the project item-list results in Step 6.
   - Update the item status:
     ```
     gh api graphql -f query='
       mutation($projectId: ID!, $itemId: ID!, $fieldId: ID!, $optionId: String!) {
         updateProjectV2ItemFieldValue(input: {
           projectId: $projectId
           itemId: $itemId
           fieldId: $fieldId
           value: { singleSelectOptionId: $optionId }
         }) {
           projectV2Item { id }
         }
       }
     ' -f projectId="<project_id>" -f itemId="<item_id>" -f fieldId="<status_field_id>" -f optionId="<in_progress_option_id>"
     ```
7. Report: "Started workspace for #N: <title> (repo: <repo>, branch: <branch>)"

## Step 9: Report Summary

Output a summary with these sections:

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
