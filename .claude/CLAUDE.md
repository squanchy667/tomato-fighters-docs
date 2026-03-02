# Tomato Fighters — Documentation Repository

This is the **documentation repo** for TomatoFighters. It contains task specs, design docs, and architecture references.

## Where to Work

All development tools (commands, agents, skills) live in the **code repo**:
```
../tomato-fighters/.claude/    ← 12 commands, 20 agents, 4 skills
```

Open Claude Code in `tomato-fighters/` (the sibling directory), not here.

## Cross-Reference

For docs↔code navigation (which task spec maps to which source file), see:
```
../tomato-fighters/.claude/CROSS_REFERENCE.md
```

## Directory Structure

```
tomato-fighters-docs/                         ← Docs repo root (GitBook)
├── .claude/CLAUDE.md                         ← This file
├── README.md                                 ← Project overview
├── PLAN.md                                   ← Architecture vision and phase outline
├── SUMMARY.md                                ← GitBook navigation (table of contents)
├── TASK_BOARD.md                             ← Master: 60 tasks, 6 phases, dependencies
├── TASK_LOGBOOK.md                           ← Execution history and session logs
├── development-agents.md                     ← Agent strategy and batch plan
├── architecture/
│   ├── system-overview.md                    → code: all Scripts/ pillars
│   ├── interface-contracts.md                → code: Scripts/Shared/Interfaces/
│   └── data-flow.md                          → code: Scripts/Shared/Data/, Enums/
├── developer/
│   ├── setup-guide.md                        ← Environment setup
│   ├── coding-standards.md                   ← Naming, patterns, rules
│   ├── workflow-guide.md                     ← Git, task execution flow
│   ├── dev1-combat-guide.md                  → code: Scripts/Combat/, Characters/
│   ├── dev2-roguelite-guide.md               → code: Scripts/Roguelite/, Paths/
│   └── dev3-world-guide.md                   → code: Scripts/World/
├── design-specs/
│   ├── CHARACTER-ARCHETYPES.md               → code: ScriptableObjects/Characters/
│   ├── PROJECT-TALAMH-CHARACTERIZATION.md    ← Game design reference
│   ├── AGENTPILOT-BOOTSTRAP.md               ← AgentPilot integration
│   └── agent-pilot-guide-*.md                ← Agent execution guides
├── product/
│   ├── features.md                           ← Feature descriptions
│   └── roadmap.md                            ← Release timeline
├── resources/
│   ├── tech-stack.md                         ← Unity, C#, packages
│   ├── changelog.md                          ← Version history
│   └── known-issues.md                       ← Open bugs/blockers
├── testing/
│   └── test-plan.md                          → code: Tests/EditMode/
└── tasks/phase-1/
    ├── T001-shared-contracts.md              → code: Scripts/Shared/
    ├── T002-character-controller.md          → code: Scripts/Combat/Movement/
    ├── T003-input-buffer.md                  → code: Scripts/Combat/Combo/
    ├── T004-combo-chain.md                   → code: Scripts/Combat/Combo/
    ├── T005-attack-data-so.md                → code: ScriptableObjects/Attacks/
    ├── T006-character-base-stats.md          → code: ScriptableObjects/Characters/
    ├── T007-stat-calculator.md               → code: Scripts/Paths/
    ├── T008-path-data-so.md                  → code: Scripts/Shared/Data/PathData.cs
    ├── T009-currency-manager.md              → code: Scripts/Roguelite/
    ├── T010-wave-manager.md                  → code: Scripts/World/
    ├── T011-enemy-base.md                    → code: Scripts/World/
    ├── T012-camera-controller.md             → code: Scripts/World/
    └── T013-test-scene.md                    → code: Scenes/
```

### Sibling Code Repo

```
../tomato-fighters/                           ← Code repo
├── .claude/                                  ← 13 commands, 20 agents, 4 skills
│   └── CROSS_REFERENCE.md                    ← Full docs↔code mapping tables
└── unity/TomatoFighters/Assets/
    ├── Scripts/
    │   ├── Shared/                           ← ALL devs (interfaces, enums, data)
    │   ├── Combat/                           ← Dev 1 (combo, movement)
    │   ├── Characters/                       ← Dev 1 (input handling)
    │   ├── Paths/                            ← Dev 2 (stat calculation)
    │   ├── Roguelite/                        ← Dev 2 (pending)
    │   └── World/                            ← Dev 3 (pending)
    ├── ScriptableObjects/                    ← Character stats, combos, movement configs
    ├── Scenes/                               ← MovementTest, SampleScene
    └── Tests/EditMode/                       ← Combo + Movement tests
```

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
