# GitHub Projects Orchestrator Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a `github-orchestrator` plugin and rename/consolidate existing VK skills, so users can orchestrate from GitHub Projects or Vibe Kanban boards.

**Architecture:** Two plugins in the marketplace — `vk-orchestrator` (consolidated from existing `orchestrator` + `plan-to-board`) and `github-orchestrator` (new). Both share `~/.vibe-kanban-orchestrate.json` for execution config. GitHub variant uses `gh` CLI for issue/project management and Vibe Kanban workspaces for agent execution.

**Tech Stack:** Markdown skills, JSON plugin config, `gh` CLI, GitHub GraphQL API

**Design doc:** `docs/plans/2026-03-17-github-projects-orchestrator-design.md`

---

### Task 1: Consolidate VK plugins and rename skills

Merge the two existing plugins (`orchestrator` and `plan-to-board`) into a single `vk-orchestrator` plugin with renamed skills.

**Files:**
- Create: `plugins/vk-orchestrator/.claude-plugin/plugin.json`
- Create: `plugins/vk-orchestrator/skills/vk-orchestrate/SKILL.md`
- Create: `plugins/vk-orchestrator/skills/plan-to-vk-project/SKILL.md`
- Delete: `plugins/orchestrator/` (entire directory)
- Delete: `plugins/plan-to-board/` (entire directory)
- Modify: `.claude-plugin/marketplace.json`

**Step 1: Create the new plugin.json**

Create `plugins/vk-orchestrator/.claude-plugin/plugin.json`:

```json
{
  "name": "vk-orchestrator",
  "description": "Orchestration and plan-to-board skills for Vibe Kanban — checks for open tasks, manages workspace lifecycle, starts new workspaces, and creates issues from plans.",
  "version": "0.1.0",
  "author": {
    "name": "Chris Banes",
    "email": "chris@banes.me"
  },
  "homepage": "https://chrisbanes.me",
  "repository": "https://github.com/chrisbanes/vibe-kanban-orchestrator",
  "license": "Apache-2.0",
  "keywords": ["orchestration", "vibe-kanban", "automation", "task-management", "plan"]
}
```

**Step 2: Move and rename the orchestrate skill**

Copy `plugins/orchestrator/skills/orchestrator/SKILL.md` to `plugins/vk-orchestrator/skills/vk-orchestrate/SKILL.md`.

Update the frontmatter `name` field from `orchestrate` to `vk-orchestrate`. Update the `description` to clarify it's the Vibe Kanban variant. Keep all other content identical.

**Step 3: Move and rename the plan-to-board skill**

Copy `plugins/plan-to-board/skills/plan-to-board/SKILL.md` to `plugins/vk-orchestrator/skills/plan-to-vk-project/SKILL.md`.

Update the frontmatter `name` field from `plan-to-board` to `plan-to-vk-project`. Update the `description` to clarify it's the Vibe Kanban variant. Update the final report section to say `Run /vk-orchestrate` instead of `Run /orchestrator`. Keep all other content identical.

**Step 4: Delete old plugin directories**

Delete `plugins/orchestrator/` and `plugins/plan-to-board/` entirely.

**Step 5: Update marketplace.json**

Update `.claude-plugin/marketplace.json` — replace the two existing plugin entries with one `vk-orchestrator` entry:

```json
{
  "name": "vibe-kanban-orchestrator",
  "owner": {
    "name": "Chris Banes",
    "email": "chris@banes.me"
  },
  "metadata": {
    "description": "Plugins for Vibe Kanban — AI-driven task management and orchestration for Claude Code.",
    "version": "0.2.0"
  },
  "plugins": [
    {
      "name": "vk-orchestrator",
      "source": "./plugins/vk-orchestrator",
      "description": "Orchestration and plan-to-board skills for Vibe Kanban — checks for open tasks, manages workspace lifecycle, starts new workspaces, and creates issues from plans.",
      "version": "0.1.0",
      "author": {
        "name": "Chris Banes",
        "email": "chris@banes.me"
      },
      "homepage": "https://chrisbanes.me",
      "repository": "https://github.com/chrisbanes/vibe-kanban-orchestrator",
      "license": "Apache-2.0",
      "keywords": ["orchestration", "vibe-kanban", "automation", "task-management", "plan"]
    }
  ]
}
```

**Step 6: Verify structure**

Run: `find plugins/ -type f | sort`

Expected:
```
plugins/vk-orchestrator/.claude-plugin/plugin.json
plugins/vk-orchestrator/skills/plan-to-vk-project/SKILL.md
plugins/vk-orchestrator/skills/vk-orchestrate/SKILL.md
```

**Step 7: Commit**

```bash
git add -A
git commit -m "refactor: consolidate VK plugins and rename skills

Merge orchestrator and plan-to-board into single vk-orchestrator plugin.
Rename skills: orchestrate → vk-orchestrate, plan-to-board → plan-to-vk-project."
```

---

### Task 2: Create `gh-orchestrate` skill

Write the GitHub Projects variant of the orchestration skill.

**Files:**
- Create: `plugins/github-orchestrator/.claude-plugin/plugin.json`
- Create: `plugins/github-orchestrator/skills/gh-orchestrate/SKILL.md`

**Step 1: Create the plugin.json**

Create `plugins/github-orchestrator/.claude-plugin/plugin.json`:

```json
{
  "name": "github-orchestrator",
  "description": "Orchestration and plan-to-project skills for GitHub Projects — checks for open tasks, manages workspace lifecycle, starts new workspaces, and creates issues from plans.",
  "version": "0.1.0",
  "author": {
    "name": "Chris Banes",
    "email": "chris@banes.me"
  },
  "homepage": "https://chrisbanes.me",
  "repository": "https://github.com/chrisbanes/vibe-kanban-orchestrator",
  "license": "Apache-2.0",
  "keywords": ["orchestration", "github-projects", "automation", "task-management"]
}
```

**Step 2: Write the gh-orchestrate SKILL.md**

Create `plugins/github-orchestrator/skills/gh-orchestrate/SKILL.md` with the full skill content.

Use the design doc (`docs/plans/2026-03-17-github-projects-orchestrator-design.md`, "gh-orchestrate Skill Flow" section) as the source of truth. The skill should:

- Frontmatter: `name: gh-orchestrate`, description referencing GitHub Projects
- Step 1: Load config from `~/.vibe-kanban-orchestrate.json` (including `github.project_number` and `github.owner`)
- Step 2: Check Completed Work — list VK workspaces, parse `#N` from name, `gh issue view N` to check if closed, archive workspace if so
- Step 3: Check for Reviewable Work — find open PRs via `gh pr list`, start review session, keep status "In Progress"
- Step 4: Check Workspace Health — stale/failed workspace detection
- Step 5: Check Concurrency — count active workspaces vs max
- Step 6: Gather Eligible Work — `gh project item-list` filtered to "Todo"
- Step 7: Filter by Dependencies — `gh api` GraphQL to check sub-issues
- Step 8: Start Workspace — `start_workspace` with name `#N Title`, move project item to "In Progress" via `gh project item-edit`
- Step 9: Report summary

For each `gh` command, write the exact command with flags. For GraphQL queries, write the full query string.

**Step 3: Commit**

```bash
git add plugins/github-orchestrator/
git commit -m "feat: add gh-orchestrate skill for GitHub Projects"
```

---

### Task 3: Create `plan-to-gh-project` skill

Write the GitHub Projects variant of the plan-to-board skill.

**Files:**
- Create: `plugins/github-orchestrator/skills/plan-to-gh-project/SKILL.md`

**Step 1: Write the plan-to-gh-project SKILL.md**

Create `plugins/github-orchestrator/skills/plan-to-gh-project/SKILL.md` with the full skill content.

Use the design doc ("plan-to-gh-project Skill Flow" section) as the source of truth. The skill should:

- Frontmatter: `name: plan-to-gh-project`, description referencing GitHub Projects
- Step 1: Load config from `~/.vibe-kanban-orchestrate.json` (including `github.project_number`, `github.owner`, `plan_directory`)
- Step 2: Find plans — glob for `{plan_directory}/*.md`
- Step 3: Select plan — if multiple, ask user
- Step 4: Extract title — first `# ` heading
- Step 5: Determine repo — if in a git repo use it (`gh repo view --json nameWithOwner`), otherwise list repos and ask
- Step 6: Set priority — scan for keywords, map to GitHub labels (`priority:urgent`, etc.), ask user to confirm
- Step 7: Create parent issue — `gh issue create`, add to project via `gh project item-add`, set status to "Todo"
- Step 8: Create sub-issues — parse `### Task` sections, `gh issue create` for each, link as sub-issue via `gh api` GraphQL, add to project
- Step 9: Report summary

For the GraphQL sub-issue linking, use:
```graphql
mutation {
  addSubIssue(input: { issueId: "<parent_node_id>", subIssueId: "<child_node_id>" }) {
    issue { id }
    subIssue { id }
  }
}
```

To get node IDs, parse from `gh issue create --json id` or `gh issue view N --json id`.

**Step 2: Commit**

```bash
git add plugins/github-orchestrator/skills/plan-to-gh-project/
git commit -m "feat: add plan-to-gh-project skill for GitHub Projects"
```

---

### Task 4: Update marketplace.json with github-orchestrator

**Files:**
- Modify: `.claude-plugin/marketplace.json`

**Step 1: Add github-orchestrator to marketplace**

Add a second entry to the `plugins` array in `.claude-plugin/marketplace.json`:

```json
{
  "name": "github-orchestrator",
  "source": "./plugins/github-orchestrator",
  "description": "Orchestration and plan-to-project skills for GitHub Projects — checks for open tasks, manages workspace lifecycle, starts new workspaces, and creates issues from plans.",
  "version": "0.1.0",
  "author": {
    "name": "Chris Banes",
    "email": "chris@banes.me"
  },
  "homepage": "https://chrisbanes.me",
  "repository": "https://github.com/chrisbanes/vibe-kanban-orchestrator",
  "license": "Apache-2.0",
  "keywords": ["orchestration", "github-projects", "automation", "task-management"]
}
```

**Step 2: Verify final structure**

Run: `find plugins/ -type f | sort`

Expected:
```
plugins/github-orchestrator/.claude-plugin/plugin.json
plugins/github-orchestrator/skills/gh-orchestrate/SKILL.md
plugins/github-orchestrator/skills/plan-to-gh-project/SKILL.md
plugins/vk-orchestrator/.claude-plugin/plugin.json
plugins/vk-orchestrator/skills/plan-to-vk-project/SKILL.md
plugins/vk-orchestrator/skills/vk-orchestrate/SKILL.md
```

**Step 3: Commit**

```bash
git add .claude-plugin/marketplace.json
git commit -m "feat: add github-orchestrator to marketplace"
```

---

### Task 5: Update README

**Files:**
- Modify: `README.md`

**Step 1: Update README**

Update the README to reflect:
- Two plugins available: `vk-orchestrator` and `github-orchestrator`
- Updated install commands for both
- Brief description of each skill (`vk-orchestrate`, `plan-to-vk-project`, `gh-orchestrate`, `plan-to-gh-project`)
- Config section showing the `github` config block
- Note that `gh` CLI must be installed and authenticated for GitHub variant

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: update README for new plugin structure"
```
