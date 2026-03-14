# Plan-to-Board Skill Design

**Goal:** A new skill that reads a plan document and creates a Vibe Kanban issue from it, bridging the gap between plan authoring and orchestrated execution.

**Architecture:** A single SKILL.md file at `skills/plan-to-board/SKILL.md` that uses Vibe Kanban MCP tools to create issues from plan files. Config lives in the existing `~/.vibe-kanban-orchestrate.json`.

## Flow

1. **Load config** — read `~/.vibe-kanban-orchestrate.json`, get `plan_directory` (default: `docs/plans/`).
2. **Find plans** — glob `{plan_directory}/*.md` for plan files.
3. **Select plan** — if one plan, use it. If multiple, list them and ask the user to pick. If none, report and stop.
4. **Determine project** — call `get_context`. If a project is available, use it. Otherwise, `list_organizations` → `list_projects` → ask user to pick.
5. **Infer priority** — scan the plan content for urgency signals (e.g., "critical", "urgent", "high priority", "blocking"), suggest a default. Ask the user to confirm/change: urgent, high, medium, low.
6. **Create issue** — `create_issue(title: <plan heading>, description: <full plan content>, project_id, priority)`.
7. **Offer to update orchestrator prompt** — check the current `prompt` in config. If it doesn't reference following the issue description as a plan, offer to update it (e.g., "Read the issue description carefully — it contains your implementation plan. Implement it step by step, following existing conventions. Create a PR when done."). User confirms or edits before writing.
8. **Report** — output the created issue ID/simple_id, title, and a reminder to invoke `/orchestrator` or wait for the next cron run.

## Config

New optional field in `~/.vibe-kanban-orchestrate.json`:

```json
{
  "plan_directory": "docs/plans/"
}
```

## What it doesn't do

- No sub-issues or blocking relationships — the plan is a single ticket
- No assumptions about what skills/workflows the workspace agent uses
- No modification of the plan file itself
- No automatic orchestrator invocation

## Invocation

`/plan-to-board` in Claude Code.

## Files to create/modify

- Create: `skills/plan-to-board/SKILL.md`
- Modify: `README.md` (add `/plan-to-board` documentation)
