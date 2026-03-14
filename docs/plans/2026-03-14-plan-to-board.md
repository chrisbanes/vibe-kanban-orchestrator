# Plan-to-Board Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a `/plan-to-board` skill that reads a plan document and creates a Vibe Kanban issue from it.

**Architecture:** A new SKILL.md file following the same pattern as the existing orchestrator skill — YAML frontmatter + numbered steps using Vibe Kanban MCP tools. README updated to document the new skill and config field.

**Tech Stack:** Markdown (SKILL.md skill definition)

---

### Task 1: Create the plan-to-board SKILL.md

**Files:**
- Create: `skills/plan-to-board/SKILL.md`

**Step 1: Create the skill file**

Create `skills/plan-to-board/SKILL.md` with this content:

````markdown
---
name: plan-to-board
description: Read a plan document and create a Vibe Kanban issue from it. Use after writing a plan to queue it for orchestrated execution.
---

# Plan to Board

You are creating a Vibe Kanban issue from a plan document. Follow these steps exactly.

## Step 1: Load Config

Read `~/.vibe-kanban-orchestrate.json` to get:
- `plan_directory` (default: `docs/plans/`)

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

## Step 7: Create Issue

1. Call `create_issue` with:
   - `title`: The extracted title from Step 4
   - `description`: The entire plan file content verbatim
   - `project_id`: From Step 5
   - `priority`: From Step 6
2. Report: "Created issue <simple_id>: <title>"

## Step 8: Offer Prompt Update

1. Read `~/.vibe-kanban-orchestrate.json` again to check the current `prompt` field.
2. Ask the user: "Would you like to update the orchestrator prompt to tell workspace agents to follow plan-based issue descriptions? Current prompt: `<current prompt>`"
3. If the user agrees, suggest this prompt:
   - "Read the issue description carefully — it contains your implementation plan. Implement it step by step, following existing conventions. Create a PR when done."
4. Let the user confirm or edit the prompt before writing.
5. If confirmed, update `~/.vibe-kanban-orchestrate.json` with the new `prompt` value, preserving all other fields.
6. If the user declines, skip this step.

## Step 9: Report

Output a summary:

```
## Plan to Board

- **Issue:** <simple_id> — <title>
- **Project:** <project name>
- **Priority:** <priority>
- **Prompt updated:** Yes/No

Run `/orchestrator` to pick up this issue, or wait for the next scheduled run.
```
````

**Step 2: Commit**

```bash
git add skills/plan-to-board/SKILL.md
git commit -m "feat: add plan-to-board skill"
```

---

### Task 2: Update README

**Files:**
- Modify: `README.md`

**Step 1: Add plan-to-board documentation to README**

After the existing "What it does" section (line 63), add a new section for `/plan-to-board`. Also add the `plan_directory` field to the configuration table at line 44.

Add to the config JSON example after the `"review"` block:

```json
  "plan_directory": "docs/plans/"
```

Add to the config table:

```markdown
| `plan_directory` | `"docs/plans/"` | Directory to search for plan files |
```

Add a new section after "What it does":

```markdown
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
```

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add plan-to-board to README"
```
