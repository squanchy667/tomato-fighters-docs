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

### T012: CameraController2D [DONE]
- **Date:** 2026-03-05
- **Dev:** Dev 3
- **Agent/Command Used:** /task-execute
- **Context Files:** `Shared/Events/VoidEventChannel.cs`, `World/WaveManager.cs`, `World/LevelBound.cs`
- **Output Files:** `World/CameraController2D.cs`
- **Issues:** None.
- **Lesson Learned:** SO event channels (VoidEventChannel) enable clean camera integration — camera subscribes to lock/unlock/stun events without referencing Combat or World systems directly.
- **Deliverables:**
  - CameraController2D with smooth follow, leading, bounds clamping, stun zoom, co-op framing
  - All configuration via Inspector (SerializeField + Tooltips + Range attributes)
  - Gizmo visualization for level bounds

### T010: WaveManager [DONE]
- **Date:** 2026-03-05
- **Dev:** Dev 3
- **Agent/Command Used:** /task-execute
- **Context Files:** `Shared/Interfaces/IRunProgressionEvents.cs`, `Shared/Data/EnemySpawnData.cs`, `World/EnemyBase.cs`
- **Output Files:** `World/WaveManager.cs`, `World/LevelBound.cs`, `Shared/Data/WaveData.cs`, `Shared/Data/EnemySpawnData.cs`, `Shared/Events/VoidEventChannel.cs`, `Shared/Events/IntEventChannel.cs`, `Editor/WaveManagerAssetsCreator.cs`
- **Issues:** None.
- **Lesson Learned:** SO event channels (VoidEventChannel, IntEventChannel) are a clean cross-pillar communication pattern — decouple sender/receiver without interface coupling.
- **Deliverables:**
  - WaveManager with configurable wave spawning and camera stops
  - LevelBound trigger zones for camera boundaries
  - SO event channels for cross-pillar wave events
  - Editor script for asset creation

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

### T014: ComboSystem — All 4 Characters [DONE]
- **Date:** 2026-03-03
- **Dev:** Dev 1
- **Agent/Command Used:** /plan-task → manual execution
- **Context Files:** `Combat/Combo/*.cs`, `Shared/Data/AttackData.cs`, `Editor/CreateMysticaAttacks.cs`, `CHARACTER-ARCHETYPES.md`
- **Output Files:** `Editor/CreateAllCharacterAttacks.cs`, `Editor/CreateComboDefinitions.cs`, 22 AttackData .asset files, 3 ComboDefinition .asset files, 4 ComboInteractionConfig .asset files
- **Issues:** None. Pure data authoring task — no infrastructure changes needed.
- **Lesson Learned:** When the combo system is well-designed (T004), extending to all characters is a data-only task. Editor scripts for asset creation are essential since CLI can't create SOs directly.
- **Deliverables:**
  - Brutor: 7 AttackData assets (ShieldBash1-2, Sweep, Launcher, LauncherSlam, OverheadSlam, GroundPound)
  - Slasher: 8-step combo tree with Heavy→Light re-entry (unique), 8 AttackData assets
  - Mystica: 5-step combo tree with widest windows (0.4s), 1 new AttackData (EmpoweredBolt)
  - Viper: 6-step combo tree, can move during normal attacks, 6 AttackData assets
  - 4 ComboInteractionConfigs with per-character cancel/lock tuning

### T023: Enemy Attack Patterns + Telegraphs [DONE]
- **Date:** 2026-03-05
- **Dev:** Dev 3
- **Agent/Command Used:** /task-execute T023
- **Context Files:** `World/EnemyAI.cs`, `World/EnemyData.cs`, `World/States/AttackState.cs`, `World/EnemyBase.cs`, `Editor/Prefabs/BasicEnemyPrefabCreator.cs`, `Shared/Data/AttackData.cs`
- **Output Files:** `World/EnemyAttackPattern.cs` (new), `World/PatternSelector.cs` (new), `World/TelegraphVisualController.cs` (new), `Tests/EditMode/World/EnemyAttackPatternTests.cs` (new), modified EnemyAI, EnemyData, AttackState, BasicEnemyPrefabCreator, test asmdef
- **Issues:** None. Clean implementation following design decisions from /plan-task.
- **Lesson Learned:** Extracting selection logic into a pure static class (PatternSelector) keeps the MonoBehaviour thin and makes edit-mode testing straightforward without needing runtime GameObjects.
- **Deliverables:**
  - EnemyAttackPattern SO with flat AttackPatternStep[] sequence, weighted selection, range/cooldown conditions
  - TelegraphVisualController: Normal (white→yellow ramp) vs Unstoppable (red flash blink)
  - AttackState rewritten for multi-step pattern execution with ShouldAbort() early-exit
  - 3 concrete patterns for BasicMeleeEnemy: Quick Jabs (2-hit), Slash (single), Heavy Slam (unstoppable)
  - 12 edit-mode tests for PatternSelector logic

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
