# Developer Workflow Guide

How to use the toolkit to plan, execute, and document tasks.

---

## The Command Chain

```
/plan-task T001          Plan it (interactive conversation)
    │
    ▼
  Design Decisions written to task spec
    │
    ▼
/task-execute T001       Execute it (follows the spec + decisions)
    │
    ▼
  Code written to Assets/Scripts/
    │
    ▼
/check-pillar            Verify no cross-pillar violations
    │
    ▼
/sync-docs               Update TASK_BOARD.md statuses + changelog
```

---

## Step-by-Step Workflow

### 1. Pick Your Task

Check `TASK_BOARD.md` for your assigned tasks. Each developer has a crew guide:
- **Dev 1 (Combat):** `developer/dev1-combat-guide.md`
- **Dev 2 (Roguelite):** `developer/dev2-roguelite-guide.md`
- **Dev 3 (World):** `developer/dev3-world-guide.md`

### 2. Plan Before You Build

```
/plan-task T014
```

This starts an interactive conversation:
- Loads the task spec, design docs, and existing code
- Walks you through what needs to be built
- Surfaces design decisions and tradeoffs
- Asks you questions ONE AT A TIME
- Documents agreed decisions in the task spec under `## Design Decisions`

**Why this matters:** The decisions you agree on here become BINDING for execution. The execution agent reads them and follows them exactly.

### 3. Execute the Task

```
/task-execute T014
```

This runs the task autonomously:
- Reads the task spec (including your Design Decisions)
- Checks dependencies are satisfied
- Writes code following project conventions
- Verifies acceptance criteria
- Reports what was built

### 4. Verify Pillar Boundaries

```
/check-pillar combat
```

Scans your pillar's code for violations:
- No imports from other pillars
- Only `TomatoFighters.Shared.*` for cross-pillar communication

### 5. Sync Documentation

```
/sync-docs
```

Updates the docs repo:
- `TASK_BOARD.md` — marks completed tasks as DONE
- `resources/changelog.md` — adds dated entry for completed work
- `SUMMARY.md` — rebuilds navigation if new docs were added

---

## File Flow

The task spec is the **single source of truth** that connects planning to execution:

```
tomato-fighters-docs/tasks/phase-1/T001-shared-contracts.md
│
├── Written by:     /generate-task-specs
├── Refined by:     /plan-task         (adds Design Decisions)
├── Read by:        /task-execute      (follows spec + decisions)
├── Read by:        /do-task           (same as task-execute)
├── Updated by:     /sync-docs         (marks status DONE)
└── Analyzed by:    /capture-learnings (extracts patterns)
```

---

## Available Commands Reference

### Execution
| Command | What It Does |
|---------|-------------|
| `/do-task` | Execute a single task through the 8-step pipeline |
| `/task-execute TXXX` | Execute a task from the board by ID |
| `/execute-phase N` | Run all tasks in a phase with parallel batching |
| `/build-app` | Full orchestration across all phases |

### Planning
| Command | What It Does |
|---------|-------------|
| `/plan-task TXXX` | Interactive planning conversation before executing |
| `/generate-task-specs` | Generate detailed task specs from TASK_BOARD.md |

### Quality
| Command | What It Does |
|---------|-------------|
| `/check-pillar` | Verify no cross-pillar import violations |
| `/scan-repo` | Index codebase for smarter context selection |

### Documentation
| Command | What It Does |
|---------|-------------|
| `/sync-docs` | Update docs repo: statuses, changelog, SUMMARY.md |
| `/capture-learnings` | Extract patterns from completed phases |

---

## Git Workflow

1. Each task creates its own branch: `pillar1/T002-character-controller`
2. Commit format: `[Phase 1] T002: Implement CharacterController2D`
3. Never push directly to main — use integration branch
4. After task is verified, merge to integration branch

### Branch Prefixes
| Pillar | Prefix |
|--------|--------|
| Shared (ALL) | `shared/` |
| Combat (Dev 1) | `pillar1/` |
| Roguelite (Dev 2) | `pillar2/` |
| World (Dev 3) | `pillar3/` |

---

## Creating New Agents

If you need a specialized agent that doesn't exist (e.g., for UI/HUD, audio, networking):

```
Use the agent-tailor agent: "Create an agent for audio/SFX management"
```

It will:
- Read existing agent patterns
- Design a new agent definition
- Save it to `.claude/agents/`
- Suggest updates to other config files

---

## Typical Session

```
# Morning: check what's ready
Open TASK_BOARD.md, find your next PENDING task

# Plan it
/plan-task T014
> (discuss approach, agree on decisions)

# Execute it
/task-execute T014
> (code gets written)

# Verify
/check-pillar combat
> Combat: ✅ Clean

# Document
/sync-docs
> TASK_BOARD.md: T014 PENDING → DONE
> Changelog updated

# Commit and push
git add . && git commit -m "[Phase 2] T014: Combo system all characters"
git push
```
