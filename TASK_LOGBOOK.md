# Tomato Fighters — Task Logbook

> Running log of completed agent tasks, outcomes, and lessons learned.
> Updated by whichever dev completes a task. This is the team's shared learning engine.

---

## How to Log a Task

After completing any task through the AgentPilot pipeline (via `/do-task`, `/task-execute`, or `/execute-phase`), add an entry below using this template:

```markdown
### [TASK-ID]: Task Name [STATUS]
- **Date:** YYYY-MM-DD
- **Dev:** Dev 1/2/3
- **Agent/Command Used:** /do-task, /task-execute, manual
- **Context Files:** list of files the agent received
- **Iterations:** number of attempts
- **Output Files:** what was produced
- **Quality Score:** X/100 (if from pipeline)
- **Issues:** what went wrong (if anything)
- **Lesson Learned:** reusable insight for future tasks
- **Time:** estimated time saved vs manual
```

---

## Phase 1: Foundation

### T005: AttackData ScriptableObject [DONE]
- **Date:** 2026-03-03
- **Dev:** Dev 1
- **Agent/Command Used:** /task-execute
- **Context Files:** `Shared/Data/*.cs`, `Shared/Enums/TelegraphType.cs`, `Shared/Interfaces/IAttacker.cs`, `Combat/Combo/ComboStep.cs`, `Editor/Prefabs/PlayerPrefabCreator.cs`
- **Output Files:** `Shared/Data/AttackData.cs`, `Editor/CreateMysticaAttacks.cs`, updated `Combat/Combo/ComboStep.cs`, updated `Shared/Interfaces/IAttacker.cs`, 4 Mystica attack `.asset` files
- **Issues:** None. Clean implementation, no blockers.
- **Lesson Learned:** Editor scripts for asset creation are essential for CLI-based workflows — can't create SO assets without Unity Editor, so menu-driven creators bridge the gap.
- **Deliverables:**
  - AttackData SO with 5 inspector groups: Identity, Damage, Animation & Timing, Telegraph & State, Effects
  - 4 Mystica attacks: MysticaStrike1 (0.6×, 16f), MysticaStrike2 (0.8×, 18f), MysticaStrike3 (1.0×, 22f), MysticaArcaneBolt (1.4×, 30f)
  - ComboStep gains `AttackData attackData` field alongside existing `damageMultiplier` (DD-1: dual scaling)
  - IAttacker.CurrentAttack typed to `AttackData` (DD-3: resolves TODO from T001)
  - Editor script: `Tools > TomatoFighters > Create Mystica Attacks`

### T004: Basic Strike Combo Chain [DONE]
- **Date:** 2026-03-03
- **Dev:** Dev 1
- **Agent/Command Used:** /task-execute, manual
- **Context Files:** `Combat/Combo/ComboStateMachine.cs`, `Combat/Combo/ComboController.cs`, `Combat/Combo/ComboStep.cs`, `Combat/Combo/ComboDefinition.cs`, `Combat/Combo/ComboInteractionConfig.cs`, `Combat/Combo/ComboDebugUI.cs`, `Combat/Movement/CharacterMotor.cs`, `Characters/CharacterInputHandler.cs`
- **Output Files:** 7 code files + 1 test file (44 unit tests) + editor updates + Brutor_ComboDefinition asset
- **Issues:** InputBufferSystem integration checkbox was misleading — T003 was implemented as a 1-slot buffer *inside* ComboStateMachine, not as a standalone system. The checkbox "consume from T003 instead of internal buffer" implied they were separate systems, but they are the same (DD-10). This caused the task to appear incomplete when it was functionally done.
- **Lesson Learned:** When a dependency task (T003) changes scope during implementation (standalone → embedded), update the dependent task's (T004) acceptance criteria immediately. Stale cross-references between tasks create phantom blockers.
- **Deliverables:**
  - Branching 7-step combo tree for Brutor (L→L→L sweep, L→H launcher, H→H ground pound)
  - Hit-confirm cancel system with dash-cancel and jump-cancel
  - ComboInteractionConfig SO for per-character cancel/reset/movement-lock tuning
  - Movement lock integration (CharacterMotor.SetAttackLock)
  - ForceResetCombo on stagger/death
  - Cancel input buffering in ComboController with priority resolution
  - 44 edit-mode unit tests (29 core + 15 hit-confirm/cancel)

---

## Phase 2: Core Combat + Path Framework

*(Pending Phase 1 completion)*

---

## Phase 3: Defensive Depth + Build Crafting

*(Pending Phase 2 completion)*

---

## Phase 4: Advanced Combat + Meta-Progression

*(Pending Phase 3 completion)*

---

## Phase 5: Content + Co-op

*(Pending Phase 4 completion)*

---

## Phase 6: Polish + Full Loop

*(Pending Phase 5 completion)*

---

## Cross-Phase Lessons

> Patterns that emerge across multiple phases. Promote here when a lesson appears 3+ times.

*(No entries yet)*

---

## Config Bank Updates

> Track when AgentPilot configs are modified based on task outcomes.

| Date | Config Modified | Change | Reason |
|------|----------------|--------|--------|
| — | — | — | — |
