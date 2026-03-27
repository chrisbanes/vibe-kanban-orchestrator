---
name: plan-to-project
description: Read a plan document and create a Vibe Kanban issue from it. Use after writing a plan to queue it for orchestrated execution.
---

# Plan to Board

You are creating a Vibe Kanban issue from a plan document. Follow these steps exactly.

## Step 1: Load Config

Read `~/.vibe-kanban-orchestrate.json` to get:
- `plan_directory` (default: `docs/plans`)

If the file doesn't exist or is invalid, use the default.

## Step 2: Find Plans

1. Glob for `{plan_directory}/*.md` to find plan files.
2. If no plan files found, report "No plan files found in {plan_directory}" and stop.

## Step 3: Select Plan

1. If only one plan file exists, use it.
2. If multiple plan files exist, list them with their filenames and the first heading from each file. Ask the user to pick one.
3. Read the selected plan file.

## Step 4: Extract Title

1. Find the first `# ` heading in the plan content. Use it as the issue title.
2. If no heading found, use the filename (without date prefix and extension) as the title.

## Step 5: Determine Project

1. Call `get_context` to check for a project in the current MCP context.
2. If a `project_id` is available, use it.
3. Otherwise:
   - Call `list_organizations` to get all orgs.
   - For each org, call `list_projects(organization_id)`.
   - If only one project exists, use it.
   - If multiple projects exist, list them and ask the user to pick one.

## Step 6: Set Priority

1. Scan the plan content for priority signals:
   - Words like "critical", "urgent", "immediately", "asap" → suggest `urgent`
   - Words like "important", "high priority", "blocking" → suggest `high`
   - Words like "low priority", "nice to have", "when possible" → suggest `low`
   - Otherwise → suggest `medium`
2. Present the suggested priority and ask the user to confirm or change it.
   - Options: urgent, high, medium, low

## Step 7: Choose Issue Granularity

Ask the user:

```
How should the plan be added to the board?

1. Single issue — the entire plan becomes one issue
2. One issue per task — each ### Task in the plan becomes a separate issue

Which option?
```

## Step 8: Create Issues

### If Option 1 (Single issue):

1. Call `create_issue` with:
   - `title`: The extracted title from Step 4
   - `description`: The entire plan file content verbatim
   - `project_id`: From Step 5
   - `priority`: From Step 6
2. Report: "Created issue <simple_id>: <title>"

### If Option 2 (One issue per task):

1. Parse the plan for headings that start with `### Task` (e.g., `### Task 1: Component Name`, `### Task 2: API Layer`). Each task section runs from its heading to the next `### Task` heading (or end of file).
2. Create a parent issue:
   - `title`: The extracted title from Step 4
   - `description`: The plan header content (everything before the first `### Task`)
   - `project_id`: From Step 5
   - `priority`: From Step 6
3. For each task section, call `create_issue` with:
   - `title`: The task heading text (e.g., "Task 1: Component Name")
   - `description`: The full task section content verbatim
   - `project_id`: From Step 5
   - `priority`: From Step 6
   - `parent_issue_id`: The parent issue ID from step 2
4. Report each created issue: "Created issue <simple_id>: <title>"

## Step 9: Offer Prompt Update

1. Read `~/.vibe-kanban-orchestrate.json` again to check the current `prompt` field.
2. Ask the user: "Would you like to update the orchestrator prompt to tell workspace agents to follow plan-based issue descriptions? Current prompt: `<current prompt>`"
3. If the user agrees, suggest this prompt:
   - "Read the issue description carefully — it contains your implementation plan. Implement it step by step, following existing conventions. Create a PR when done."
4. Let the user confirm or edit the prompt before writing.
5. If confirmed, update `~/.vibe-kanban-orchestrate.json` with the new `prompt` value, preserving all other fields.
6. If the user declines, skip this step.

## Step 10: Report

Output a summary:

```
## Plan to Board

- **Issues created:** <count>
- **Project:** <project name>
- **Priority:** <priority>
- **Prompt updated:** Yes/No

### Issues
- <simple_id> — <title>
- <simple_id> — <title> (if multiple)
- ...

Run `/orchestrate` to pick up these issues, or wait for the next scheduled run.
```
