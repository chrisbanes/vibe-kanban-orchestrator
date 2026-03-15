# Split into Two Plugins

## Goal

Split the single `vibe-kanban-orchestrator` plugin into two independently installable plugins — `orchestrator` and `plan-to-board` — within the same repository, both listed in the marketplace.

## Motivation

Some users only want the `/orchestrate` skill for workspace lifecycle management and don't need `/plan-to-board`. Splitting lets users install only what they need.

## Design

### Repository Structure

```
vibe-kanban-orchestrator/
  .claude-plugin/
    marketplace.json          ← lists both plugins
  plugins/
    orchestrator/
      .claude-plugin/
        plugin.json
      skills/
        orchestrator/
          SKILL.md            ← moved from skills/orchestrator/
    plan-to-board/
      .claude-plugin/
        plugin.json
      skills/
        plan-to-board/
          SKILL.md            ← moved from skills/plan-to-board/
```

### Marketplace

`marketplace.json` lists two plugins with relative path sources:

```json
{
  "name": "vibe-kanban-orchestrator",
  "plugins": [
    { "name": "orchestrator", "source": "./plugins/orchestrator" },
    { "name": "plan-to-board", "source": "./plugins/plan-to-board" }
  ]
}
```

### Shared Config

Both plugins continue reading `~/.vibe-kanban-orchestrate.json`. No changes to config structure.

### Install Commands

```
/plugin install orchestrator@vibe-kanban-orchestrator
/plugin install plan-to-board@vibe-kanban-orchestrator
```

### README Updates

- Installation section updated to show both install commands
- Note that `orchestrator` is the core plugin; `plan-to-board` is optional

## What Changes

- Move `skills/orchestrator/` → `plugins/orchestrator/skills/orchestrator/`
- Move `skills/plan-to-board/` → `plugins/plan-to-board/skills/plan-to-board/`
- Add `plugin.json` for each new plugin directory
- Update `.claude-plugin/marketplace.json` to list both plugins with new sources
- Remove the top-level `.claude-plugin/plugin.json` (no longer a single plugin)
- Update README installation instructions

## What Stays the Same

- All skill content (`SKILL.md` files) — no changes
- Config file path and structure
- Marketplace name (`vibe-kanban-orchestrator`)
