# Tomato Fighters — Documentation Repository

This is the **documentation repo** for TomatoFighters. It contains task specs, design docs, and architecture references.

## Where to Work

All development tools (commands, agents, skills) live in the **code repo**:
```
../tomato-fighters/.claude/    ← 8 commands, 17 agents, 4 skills
```

Open Claude Code in `tomato-fighters/` (the sibling directory), not here.

## What's in This Repo

| Path | Content |
|------|---------|
| `TASK_BOARD.md` | 60 tasks across 6 phases with dependencies |
| `PLAN.md` | Architecture vision and phase outline |
| `tasks/phase-{N}/` | Detailed task specs (TXXX-name.md) |
| `architecture/` | System overview, interface contracts, data flow |
| `design-specs/` | Character archetypes, combat design |
| `developer/` | Setup guide, coding standards, per-dev crew guides |
| `development-agents.md` | Agent strategy and batch execution plan |
| `product/` | Features and roadmap |
| `testing/` | Test plans |

## Developer Guides

- **Dev 1 (Combat):** `developer/dev1-combat-guide.md`
- **Dev 2 (Roguelite):** `developer/dev2-roguelite-guide.md`
- **Dev 3 (World):** `developer/dev3-world-guide.md`

## Quick Start

1. Clone both repos side by side:
   ```
   git clone {code-repo-url} tomato-fighters
   git clone {docs-repo-url} tomato-fighters-docs
   ```
2. Open Claude Code in `tomato-fighters/`
3. Read your crew guide above
4. Run `/task-execute TXXX` to start your assigned tasks
