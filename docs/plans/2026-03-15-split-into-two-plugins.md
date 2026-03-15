# Split into Two Plugins Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Split the single `vibe-kanban-orchestrator` plugin into two independently installable plugins (`orchestrator` and `plan-to-board`) within the same repo.

**Architecture:** Move each skill into its own plugin directory under `plugins/`, each with its own `plugin.json`. Update `marketplace.json` to list both plugins as relative-path sources. Remove the top-level `plugin.json` since this repo is now a marketplace, not a single plugin.

**Tech Stack:** JSON (plugin manifests), Markdown (skills)

---

### Task 1: Create the `orchestrator` plugin directory

**Files:**
- Create: `plugins/orchestrator/.claude-plugin/plugin.json`
- Create: `plugins/orchestrator/skills/orchestrator/SKILL.md` (moved from `skills/orchestrator/SKILL.md`)

**Step 1: Create the directory structure**

```bash
mkdir -p plugins/orchestrator/.claude-plugin
mkdir -p plugins/orchestrator/skills/orchestrator
```

**Step 2: Copy the skill file**

```bash
cp skills/orchestrator/SKILL.md plugins/orchestrator/skills/orchestrator/SKILL.md
```

**Step 3: Create `plugin.json`**

Create `plugins/orchestrator/.claude-plugin/plugin.json`:

```json
{
  "name": "orchestrator",
  "description": "Orchestration skill for Vibe Kanban — checks for open tasks, manages workspace lifecycle, and starts new workspaces for the next highest-priority task.",
  "version": "0.1.0",
  "author": {
    "name": "Chris Banes",
    "email": "chris@banes.me"
  },
  "homepage": "https://chrisbanes.me",
  "repository": "https://github.com/chrisbanes/vibe-kanban-orchestrator",
  "license": "Apache-2.0",
  "keywords": ["orchestration", "vibe-kanban", "automation", "task-management"]
}
```

**Step 4: Verify the files exist**

```bash
ls plugins/orchestrator/.claude-plugin/plugin.json
ls plugins/orchestrator/skills/orchestrator/SKILL.md
```

Expected: both files listed.

**Step 5: Commit**

```bash
git add plugins/orchestrator/
git commit -m "feat: add orchestrator plugin directory"
```

---

### Task 2: Create the `plan-to-board` plugin directory

**Files:**
- Create: `plugins/plan-to-board/.claude-plugin/plugin.json`
- Create: `plugins/plan-to-board/skills/plan-to-board/SKILL.md` (moved from `skills/plan-to-board/SKILL.md`)

**Step 1: Create the directory structure**

```bash
mkdir -p plugins/plan-to-board/.claude-plugin
mkdir -p plugins/plan-to-board/skills/plan-to-board
```

**Step 2: Copy the skill file**

```bash
cp skills/plan-to-board/SKILL.md plugins/plan-to-board/skills/plan-to-board/SKILL.md
```

**Step 3: Create `plugin.json`**

Create `plugins/plan-to-board/.claude-plugin/plugin.json`:

```json
{
  "name": "plan-to-board",
  "description": "Read a plan document and create a Vibe Kanban issue from it. Use after writing a plan to queue it for orchestrated execution.",
  "version": "0.1.0",
  "author": {
    "name": "Chris Banes",
    "email": "chris@banes.me"
  },
  "homepage": "https://chrisbanes.me",
  "repository": "https://github.com/chrisbanes/vibe-kanban-orchestrator",
  "license": "Apache-2.0",
  "keywords": ["plan", "vibe-kanban", "task-management"]
}
```

**Step 4: Verify the files exist**

```bash
ls plugins/plan-to-board/.claude-plugin/plugin.json
ls plugins/plan-to-board/skills/plan-to-board/SKILL.md
```

Expected: both files listed.

**Step 5: Commit**

```bash
git add plugins/plan-to-board/
git commit -m "feat: add plan-to-board plugin directory"
```

---

### Task 3: Update `marketplace.json`

**Files:**
- Modify: `.claude-plugin/marketplace.json`

**Step 1: Replace the contents of `.claude-plugin/marketplace.json`**

```json
{
  "name": "vibe-kanban-orchestrator",
  "owner": {
    "name": "Chris Banes",
    "email": "chris@banes.me"
  },
  "metadata": {
    "description": "Plugins for Vibe Kanban — AI-driven task management and orchestration for Claude Code.",
    "version": "0.1.0"
  },
  "plugins": [
    {
      "name": "orchestrator",
      "source": "./plugins/orchestrator",
      "description": "Orchestration skill for Vibe Kanban — checks for open tasks, manages workspace lifecycle, and starts new workspaces for the next highest-priority task.",
      "version": "0.1.0",
      "author": {
        "name": "Chris Banes",
        "email": "chris@banes.me"
      },
      "homepage": "https://chrisbanes.me",
      "repository": "https://github.com/chrisbanes/vibe-kanban-orchestrator",
      "license": "Apache-2.0",
      "keywords": ["orchestration", "vibe-kanban", "automation", "task-management"]
    },
    {
      "name": "plan-to-board",
      "source": "./plugins/plan-to-board",
      "description": "Read a plan document and create a Vibe Kanban issue from it. Use after writing a plan to queue it for orchestrated execution.",
      "version": "0.1.0",
      "author": {
        "name": "Chris Banes",
        "email": "chris@banes.me"
      },
      "homepage": "https://chrisbanes.me",
      "repository": "https://github.com/chrisbanes/vibe-kanban-orchestrator",
      "license": "Apache-2.0",
      "keywords": ["plan", "vibe-kanban", "task-management"]
    }
  ]
}
```

**Step 2: Commit**

```bash
git add .claude-plugin/marketplace.json
git commit -m "feat: update marketplace to list orchestrator and plan-to-board plugins"
```

---

### Task 4: Remove the top-level `plugin.json` and old skill directories

**Files:**
- Delete: `.claude-plugin/plugin.json`
- Delete: `skills/orchestrator/SKILL.md`
- Delete: `skills/plan-to-board/SKILL.md`
- Delete: `skills/` directory (if now empty)

**Step 1: Remove the files**

```bash
git rm .claude-plugin/plugin.json
git rm -r skills/
```

**Step 2: Verify removal**

```bash
ls .claude-plugin/
```

Expected: only `marketplace.json` listed.

```bash
ls skills/ 2>&1
```

Expected: `ls: skills/: No such file or directory`

**Step 3: Commit**

```bash
git commit -m "chore: remove top-level plugin.json and old skills directory"
```

---

### Task 5: Update the README

**Files:**
- Modify: `README.md`

**Step 1: Update the Installation section**

Replace the Installation section with:

```markdown
## Installation

Add the marketplace, then install the plugin(s) you want:

```
/plugin marketplace add chrisbanes/vibe-kanban-orchestrator
```

Install the orchestrator skill:

```
/plugin install orchestrator@vibe-kanban-orchestrator
```

Optionally, install the plan-to-board skill:

```
/plugin install plan-to-board@vibe-kanban-orchestrator
```
```

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: update README installation instructions for split plugins"
```
