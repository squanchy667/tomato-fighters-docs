# Tomato Fighters — Development Agents & Execution Strategy

> This project uses the **AgentPilot / Project Factory** meta-commands from `/Users/ofek/Projects/Claude/.claude/`.
> All 3 developers run tasks through the same pipeline: `/do-task`, `/task-execute`, `/execute-phase`.

---

## Available Meta-Commands

These commands are already configured in the workspace `.claude/` directory:

| Command | When to Use | Who Uses It |
|---------|------------|-------------|
| `/plan-project` | Decompose a goal into phased tasks | Product Owner (Ofek) |
| `/execute-phase N` | Run all tasks in a phase with parallel execution | Any dev starting a phase |
| `/do-task` | Execute a single task through the 8-step pipeline | Any dev on their assigned task |
| `/task-execute` | Execute a task from the task board autonomously | Any dev on their assigned task |
| `/build-app` | Full orchestration (plan → execute → document → learn) | Product Owner for major milestones |
| `/scan-repo` | Index the Unity codebase for smarter context selection | After major merges |
| `/capture-learnings` | Extract patterns from completed phase | After each phase completes |
| `/design-schema` | Design a data model (Zod-style validation-first) | Dev 2 for data structures |

---

## Agent Types per Pillar

### Dev 1 — Combat Agents

| Agent Config | Model | Tasks | Token Budget |
|-------------|-------|-------|-------------|
| `unity-combat` | sonnet | Combo, hitbox, defense, pressure, wall bounce | 10K |
| `unity-combat-feel` | sonnet | Hitstop, screen shake, timing tuning | 8K |

**Context Strategy:** `combat` — sees `Combat/`, `Characters/`, `Shared/` only.

### Dev 2 — Roguelite Agents

| Agent Config | Model | Tasks | Token Budget |
|-------------|-------|-------|-------------|
| `unity-roguelite` | sonnet | Rituals, paths, stats, meta, save/load | 10K |
| `unity-path-system` | sonnet | Path selection, tier logic, stat calc | 10K |

**Context Strategy:** `roguelite` — sees `Roguelite/`, `Paths/`, `Shared/` only.

### Dev 3 — World Agents

| Agent Config | Model | Tasks | Token Budget |
|-------------|-------|-------|-------------|
| `unity-world` | sonnet | Enemies, bosses, waves, camera, UI, animation | 10K |

**Context Strategy:** `world` — sees `World/`, `Shared/`, `Animations/` only.

### Cross-Pillar Agents

| Agent Config | Model | Tasks | Token Budget |
|-------------|-------|-------|-------------|
| `unity-integration` | sonnet | Interface wiring, cross-domain testing | 12K |

**Context Strategy:** `fullstack` — sees all code, compressed.

---

## Batch Execution Plan

### Phase 1: Foundation (13 tasks)

| Batch | Dev | Tasks | Parallel? | Rationale |
|-------|-----|-------|-----------|-----------|
| 1.0 | ALL | T001 (shared interfaces) | Solo — sync session | All 3 devs must agree on contracts |
| 1.1a | Dev 1 | T002 (controller), T003 (input), T005 (attackdata) | Yes — no deps between them | Foundation combat files |
| 1.1b | Dev 2 | T006 (base stats), T009 (currency) | Yes — independent | Foundation data files |
| 1.1c | Dev 3 | T010 (waves), T011 (enemy base), T012 (camera) | Yes — independent | Foundation world files |
| 1.2a | Dev 1 | T004 (combo chain) | Solo — needs T002+T003 | Builds on controller + input |
| 1.2b | Dev 2 | T007 (stat calc), T008 (path data) | Yes — independent | Builds on base stats |
| 1.2c | Dev 3 | T013 (test scene) | Solo — needs T002+T010+T011+T012 | Integration scene |

### Phase 2: Core Combat + Paths (12 tasks)

| Batch | Dev | Tasks | Parallel? | Rationale |
|-------|-----|-------|-----------|-----------|
| 2.1a | Dev 1 | T014 (combo all chars), T015 (hitbox) | Sequential — T015 needs T014 | Combo → hitbox |
| 2.1b | Dev 2 | T018 (path system), T020 (ritual data) | Yes — independent | Path + ritual foundations |
| 2.1c | Dev 3 | T022 (enemy AI), T024 (animators) | Yes — independent | Enemy + animation |
| 2.2a | Dev 1 | T016 (defense), T017 (passives) | Yes — both need T014 | Defense systems |
| 2.2b | Dev 2 | T019 (path UI), T021 (ritual system) | Sequential — T021 needs T020 | UI + ritual pipeline |
| 2.2c | Dev 3 | T023 (enemy attacks), T025 (HUD) | Yes — independent | Enemies + UI |

### Phase 3-6: Follow dependency chains in TASK_BOARD.md

---

## Execution Workflow Per Developer

```
1. Check TASK_BOARD.md → find your next PENDING task
2. Verify dependencies are DONE
3. Run:  /do-task "T0XX: [task description]"
   or:  /task-execute  (if task is on the board)
4. Review output → iterate if needed
5. Copy output to Unity project (or auto if agent writes directly)
6. Update TASK_BOARD.md: [PENDING] → [DONE]
7. Log in TASK_LOGBOOK.md: date, agent, iterations, lessons
8. Commit: git commit -m "[Phase X] T0XX: Brief description"
9. Push to your pillar branch: pillar{N}/feature-name
```

---

## Integration Points (All 3 Devs)

| Trigger | Action | Command |
|---------|--------|---------|
| End of each batch | Merge pillar branches → integration branch | `git merge` |
| End of each phase | Full integration test | `/execute-phase N` validation |
| After each phase | Capture learnings | `/capture-learnings TomatoFighters` |
| After major merge | Re-index codebase | `/scan-repo` |
| Interface change | All 3 review PR | Manual review |

---

## Coverage Matrix

| Agent Config | Phase 1 | Phase 2 | Phase 3 | Phase 4 | Phase 5 | Phase 6 | Total |
|-------------|---------|---------|---------|---------|---------|---------|-------|
| unity-combat | 4 | 4 | 3 | 3 | 2 | 2 | **18** |
| unity-roguelite | 4 | 4 | 3 | 4 | 4 | 2 | **21** |
| unity-world | 4 | 4 | 3 | 3 | 4 | 4 | **22** |
| unity-integration | 1 | — | — | — | — | 1 | **2** |
| **Total** | **13** | **12** | **9** | **10** | **8** | **8** | **60** |

---

## Summary

- **60 tasks** across 6 phases (12 weeks)
- **3 developers** working in parallel on isolated pillars
- **6 AgentPilot configs** auto-route tasks to the right specialist
- **Shared interfaces** are the ONLY coupling point between pillars
- **Quality gates** at every phase boundary (60% pass rate, 50+ avg quality)
- All execution through existing `/do-task`, `/execute-phase`, `/build-app` commands
