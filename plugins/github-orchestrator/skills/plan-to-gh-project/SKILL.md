---
name: plan-to-gh-project
description: Read a plan document and create GitHub Issues from it — a parent issue (epic) with sub-issues for each task. Adds all issues to a GitHub Project board.
---

# Plan to GitHub Project

You are creating GitHub Issues from a plan document and adding them to a GitHub Project board. Follow these steps exactly.

## Step 1: Load Config

Read `~/.vibe-kanban-orchestrate.json` to get:
- `github.project_number` (required) — the GitHub Project number
- `github.owner` (required) — the GitHub owner (user or organization)
- `plan_directory` (default: `docs/plans`)

If the `github.project_number` or `github.owner` fields are missing, report the error and stop:

```
Error: Missing required config in ~/.vibe-kanban-orchestrate.json

Expected format:
{
  "github": {
    "owner": "<github-user-or-org>",
    "project_number": <number>
  }
}
```

## Step 2: Find Plans

1. Glob for `{plan_directory}/*.md` to find plan files.
2. If no plan files found, report "No plan files found in {plan_directory}" and stop.

## Step 3: Select Plan

1. If only one plan file exists, use it.
2. If multiple plan files exist, list them with their filenames and the first heading from each file. Ask the user to pick one.
3. Read the selected plan file.

## Step 4: Extract Title

1. Find the first `# ` heading in the plan content. Use it as the parent issue title.
2. If no heading found, use the filename (without date prefix and extension) as the title.

## Step 5: Determine Repo

1. Check if running inside a git repo by running:
   ```
   gh repo view --json nameWithOwner -q '.nameWithOwner'
   ```
2. If the command succeeds, use the returned `owner/repo` as the target repo.
3. If the command fails (not in a git repo or no remote), list repos for the configured owner:
   ```
   gh repo list <owner> --json nameWithOwner -q '.[].nameWithOwner'
   ```
   Present the list and ask the user to pick one.

## Step 6: Set Priority

1. Scan the plan content for priority signals:
   - Words like "critical", "urgent", "immediately", "asap" -> suggest `priority:urgent`
   - Words like "important", "high priority", "blocking" -> suggest `priority:high`
   - Words like "low priority", "nice to have", "when possible" -> suggest `priority:low`
   - Otherwise -> suggest `priority:medium`
2. Present the suggested priority and ask the user to confirm or change it.
   - Options: `priority:urgent`, `priority:high`, `priority:medium`, `priority:low`

## Step 7: Create Parent Issue

1. Create the parent issue using the plan title and the content before the first `### Task` heading as the body:
   ```
   gh issue create --repo <repo> --title "<title>" --body "<plan header content before first ### Task>" --label "<priority label>"
   ```
2. Capture the issue URL and number from the output.
3. Get the node ID for the parent issue:
   ```
   gh issue view <number> --repo <repo> --json id -q '.id'
   ```
4. Add the parent issue to the GitHub Project:
   ```
   gh project item-add <project_number> --owner <owner> --url <issue_url>
   ```
5. Set the item status to "Todo" on the project board. First, get the project metadata (try as organization first, fall back to user if it fails):
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
   If that returns an error, retry with `user(login: $owner)` instead of `organization(login: $owner)`.

   Then find the option ID where `name` equals "Todo" and the item ID from the `item-add` output, and set the status:
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
   ' -f projectId="<project_id>" -f itemId="<item_id>" -f fieldId="<status_field_id>" -f optionId="<todo_option_id>"
   ```
   Cache the project metadata (project ID, field ID, Todo option ID) for reuse in Step 8.
6. Report: "Created parent issue #N: <title>"

## Step 8: Create Sub-Issues

1. Parse the plan for `### Task` headings. Each task section runs from its heading to the next `### Task` heading or end of file.
2. For each task section:
   a. Create the issue:
      ```
      gh issue create --repo <repo> --title "<task heading text>" --body "<task section content>" --label "<priority label>"
      ```
   b. Capture the URL and number from the output.
   c. Get the child node ID:
      ```
      gh issue view <number> --repo <repo> --json id -q '.id'
      ```
   d. Link it as a sub-issue to the parent using GraphQL:
      ```
      gh api graphql -f query='
        mutation($parentId: ID!, $childId: ID!) {
          addSubIssue(input: { issueId: $parentId, subIssueId: $childId }) {
            issue { id }
            subIssue { id }
          }
        }
      ' -f parentId="<parent_node_id>" -f childId="<child_node_id>"
      ```
   e. Add the sub-issue to the GitHub Project:
      ```
      gh project item-add <project_number> --owner <owner> --url <issue_url>
      ```
   f. Set the item status to "Todo" using the cached project metadata from Step 7 and the same `updateProjectV2ItemFieldValue` mutation.
3. Report each: "Created sub-issue #N: <title>"

## Step 9: Report

Output a summary:

```
## Plan to GitHub Project

- **Issues created:** <count> (1 parent + N sub-issues)
- **Repo:** <repo name>
- **Project:** #<project_number>
- **Priority:** <priority label>

### Issues
- #<number> — <parent title> (parent)
  - #<number> — <task title>
  - #<number> — <task title>
  - ...

Run `/gh-orchestrate` to pick up these issues, or wait for the next scheduled run.
```
