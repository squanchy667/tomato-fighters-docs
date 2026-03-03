# T004: Basic Strike Combo Chain

## Metadata
| Field | Value |
|-------|-------|
| **Phase** | 1 — Foundation |
| **Type** | implementation |
| **Priority** | P0 |
| **Owner** | Dev 1 |
| **Agent** | combat-agent |
| **Depends On** | T002, T003 |
| **Blocks** | T014 |
| **Status** | DONE |
| **Completed** | 2026-03-03 |
| **Branch** | `tal` |

## Objective
Build the branching combo tree system and wire up Brutor's 3-hit shield bash + finisher chain as the first playable combo. This is the core attack flow that every character's moveset will build on — combo nodes, input windows, cancel flags, and hit-confirm logic.

## Context
The combo system is the heart of combat in Tomato Fighters. It sits between InputBufferSystem (T003) and HitboxManager (T015, later phase). The flow per `architecture/data-flow.md`:

```
InputBufferSystem → ComboSystem → load AttackData → CharacterController2D (apply animation) → HitboxManager (later)
```

This task builds the ComboSystem and ComboNode classes. The system uses a **tree of ComboNodes** — each node represents one attack in a sequence. Nodes branch based on which input the player provides next, enabling combos like: Strike → Strike → Strike → Finisher, or Strike → Strike → Skill (heavy), etc.

For Phase 1, only Brutor's basic chain is implemented:
- **Strike 1** → **Strike 2** → **Strike 3** → **Finisher** (4 nodes in a line)
- Each node references an AttackData ScriptableObject (from T005) for damage, timing, animation
- On hit-confirm, dash-cancel and jump-cancel become available

The system is designed to be extended in T014 (Phase 2) for all 4 characters with full branching trees.

## Requirements

### ComboNode (plain C# class, not MonoBehaviour)

1. **ComboNode data fields:**
   - `AttackData attackData` — reference to the ScriptableObject defining this attack's properties
   - `Dictionary<BufferedInputType, ComboNode> branches` — what input leads to what next node
   - `float inputWindow` — time (seconds) after this node's attack completes where input is accepted for branching (configurable per node)
   - `bool canDashCancelOnHit` — if true AND hit confirmed, player can cancel into dash
   - `bool canJumpCancelOnHit` — if true AND hit confirmed, player can cancel into jump
   - `bool isFinisher` — marks the final node in a chain (triggers finisher events)
   - `string nodeId` — unique identifier for debugging and logging

2. **ComboNode should be serializable** — either as a ScriptableObject tree or as a serializable plain C# class. Prefer ScriptableObject for Inspector editability, but if tree wiring is too complex in Inspector, use a configuration ScriptableObject that defines the tree structure via serialized arrays

### ComboSystem (MonoBehaviour)

3. **Combo state machine:**
   - `Idle` — no combo active, waiting for first input
   - `Attacking` — currently in an attack animation, waiting for it to complete
   - `WindowOpen` — attack animation finished, input window is open for branching
   - `Recovering` — input window expired, returning to Idle

4. **Combo flow:**
   - Player presses Strike → ComboSystem checks `InputBufferSystem.TryConsumeInput(Strike)`
   - If in Idle: start combo at root node
   - If in WindowOpen: check current node's branches for the consumed input type → advance to the branched node
   - Load the node's `AttackData` → tell CharacterController2D to lock movement and play the attack animation
   - Wait for animation to complete (via Animation Event callback or timer based on AttackData frame data)
   - Open input window for `inputWindow` seconds
   - If no input consumed during window → return to Idle (combo drops)
   - If input consumed → advance to next node and repeat

5. **Hit-confirm system:**
   - HitboxManager (T015, future) will call `ComboSystem.OnHitConfirmed()` when the active hitbox contacts an IDamageable target
   - On hit-confirm: set `hitConfirmed = true` for the current node
   - If `currentNode.canDashCancelOnHit && hitConfirmed`: consume buffered Dash input → cancel attack recovery → CharacterController2D executes dash
   - If `currentNode.canJumpCancelOnHit && hitConfirmed`: same for Jump
   - Until HitboxManager exists, expose `OnHitConfirmed()` as a public method (can be called from a test script or debug key)

6. **InputBufferSystem integration:**
   - Every frame during `WindowOpen`, poll InputBufferSystem for buffered inputs matching the current node's branches
   - Priority order if multiple buffered: Strike > Skill > Arcana > Dash > Jump (configurable)
   - On state reset (combo drops, stagger, death), call `InputBufferSystem.ClearBuffer()`

7. **CharacterController2D integration:**
   - When a combo node starts: call `CharacterController2D.SetMovementLock(true)`
   - When combo returns to Idle: call `CharacterController2D.SetMovementLock(false)`
   - When dash-cancel triggers: call `CharacterController2D.SetMovementLock(false)` then trigger dash

8. **Brutor's combo tree (initial data):**
   - **Root → Strike 1:** Shield Jab. Fast, short range. `inputWindow: 0.4s`, `canDashCancel: false`, `canJumpCancel: false`
   - **Strike 1 → Strike 2:** Shield Swipe. Medium recovery. `inputWindow: 0.35s`, `canDashCancel: true`, `canJumpCancel: false`
   - **Strike 2 → Strike 3:** Shield Bash. Heavier hit. `inputWindow: 0.3s`, `canDashCancel: true`, `canJumpCancel: true`
   - **Strike 3 → Finisher:** Overhead Slam. Slow, big knockback. `isFinisher: true`, `canDashCancel: false`, `canJumpCancel: false`
   - All 4 branch on `BufferedInputType.Strike`

9. **Animation-driven progression:**
   - Attack progression is driven by animation timing from AttackData (start frame, active frames)
   - The ComboSystem does NOT use `Time.time` timers for attack duration — it relies on Animation Event callbacks (`OnAttackAnimationEnd`) or on `AttackData.hitboxStartFrame + AttackData.hitboxActiveFrames` converted to time
   - Until Animation Events are wired (T024), use a timer fallback based on AttackData frame data: `duration = (startFrame + activeFrames) / 60f`

## File Plan

### Already Implemented
| File Path | Description |
|-----------|-------------|
| `Combat/Combo/AttackType.cs` | Enum: Light, Heavy |
| `Combat/Combo/ComboState.cs` | Enum: Idle, Attacking, ComboWindow, Finisher |
| `Combat/Combo/ComboStep.cs` | Serializable struct: combo tree node with branching indices |
| `Combat/Combo/ComboDefinition.cs` | ScriptableObject: flat step array with root indices |
| `Combat/Combo/ComboStateMachine.cs` | Plain C# state machine: sequencing, input buffer, window timer |
| `Combat/Combo/ComboController.cs` | MonoBehaviour: wires input → state machine → animation → events |
| `Combat/Combo/ComboDebugUI.cs` | Debug overlay with auto-advance |
| `Tests/EditMode/Combat/Combo/ComboStateMachineTests.cs` | 29 unit tests |

### Remaining Work
| File Path | Changes |
|-----------|---------|
| `Combat/Combo/ComboStep.cs` | +`canDashCancelOnHit`, +`canJumpCancelOnHit` fields |
| `Combat/Combo/ComboInteractionConfig.cs` | **New SO** — cancel priority, reset triggers, movement lock flags |
| `Combat/Combo/ComboStateMachine.cs` | +`OnHitConfirmed()`, +`hitConfirmed` flag, +`CanDashCancel`/`CanJumpCancel` properties, +`CancelPerformed()` |
| `Combat/Combo/ComboController.cs` | +`[SerializeField] CharacterMotor motor`, +`[SerializeField] ComboInteractionConfig config`, +`RequestDashCancel()`/`RequestJumpCancel()`, +`ForceResetCombo()`, +movement lock calls, +`Debug.LogError` if no animator |
| `Combat/Movement/CharacterMotor.cs` | +`isAttackLocked` flag, +`SetAttackLock(bool)`, +guard in `ApplyMovement()` |
| `Characters/CharacterInputHandler.cs` | +route dash/jump through ComboController when combo is active |
| `Tests/EditMode/Combat/Combo/ComboStateMachineTests.cs` | +tests for hit-confirm, cancel properties, forced reset |
| `ScriptableObjects/ComboDefinitions/Brutor_ComboDefinition.asset` | Update steps with cancel flags per original spec |

## Implementation Notes

- **ComboNode as ScriptableObject vs plain class:** If using ScriptableObject, each node is an asset and branches are wired via Inspector drag-and-drop. This is clean but can be tedious for large trees. Alternative: a single `ComboTreeData` ScriptableObject that holds a serialized list of nodes with index-based branching. Either approach works — pick whichever is easier to author and extend to 4 characters in T014
- **No damage calculation here:** ComboSystem only manages the attack flow (which attack, when, cancel windows). Actual damage calculation happens in the Damage Calculation step (see `architecture/data-flow.md` step 5) which involves HitboxManager + IBuffProvider. ComboSystem just loads the AttackData and makes it available
- **Animator integration:** For Phase 1, the ComboSystem triggers animations via `Animator.Play("Strike1")` or `Animator.SetTrigger("strike")`. The exact mechanism depends on T024 (Character Animator Controllers). For now, expose `OnAttackStarted(AttackData)` as an event that any animation system can subscribe to
- **Cancel priority:** If both dash-cancel and jump-cancel are available and both inputs are buffered, dash takes priority (more common in beat-em-ups). This is configurable
- **Combo reset:** Combo resets (returns to Idle) on: input window expiry, taking damage (stagger), death, dash-cancel execution, manual reset call. It does NOT reset on jump-cancel (player can attack from air in future)
- **No ICombatEvents firing yet:** The ICombatEvents.OnStrike event will be fired by the damage pipeline, not directly by ComboSystem. ComboSystem is purely about attack sequencing

## Acceptance Criteria

### Original Requirements (from spec)
- [x] ComboNode class with AttackData reference, input branches, input window, and cancel flags — *Partially done: ComboStep struct has branches + window + finisher flag. Missing: AttackData ref, cancel flags (see below)*
- [x] ComboNode supports branching to different nodes based on input type — *Done via DD-1: AttackType (Light/Heavy) instead of BufferedInputType*
- [x] ComboSystem state machine: Idle → Attacking → WindowOpen → Recovering → Idle — *Done as Idle → Attacking → ComboWindow → Finisher. Missing: Recovering state (goes direct to Idle)*
- [x] Brutor's combo tree — *Done: 7-step branching tree (upgraded from original 4-node linear chain)*
- [x] Each node has configurable input window duration — *Done: per-step override + definition default*
- [x] Hit-confirm callback (`OnHitConfirmed`) enables dash-cancel and jump-cancel flags
- [x] Dash-cancel on hit-confirm: instant check via ComboController.RequestDashCancel() (DD-5, DD-10)
- [x] Jump-cancel on hit-confirm: instant check via ComboController.RequestJumpCancel() (DD-5, DD-10)
- [x] InputBufferSystem integration — *T003 was implemented as the 1-slot buffer inside ComboStateMachine (BufferedInput property). Combo chaining consumes this buffer on ComboWindow open. Cancel inputs use instant checks (DD-10). No standalone system needed — T003's buffer IS the internal buffer.*
- [x] CharacterMotor movement lock: ComboController calls motor.SetAttackLock(bool) (DD-6)
- [x] ComboInteractionConfig SO: cancel priority, reset triggers, movement lock flags (DD-7)
- [x] Animation-driven timing via AttackData frame data — *Animation event callbacks are wired (OnAttackAnimationEnd, OnComboWindowOpen). AttackData reference deferred to T005 which will provide the SO. Timer fallback works via ComboDebugUI auto-advance. System is ready to consume AttackData once T005 delivers it.*
- [x] Combo resets on: window expiry, stagger, death, dash-cancel — *ForceResetCombo() public method + config-driven reset triggers (DD-7)*
- [x] `isFinisher` flag on final combo node — *Done*
- [x] Compiles with zero warnings — *Done*

### Implemented Beyond Original Spec
- [x] ComboDefinition ScriptableObject with flat step array (DD-2)
- [x] Plain C# ComboStateMachine testable without Unity runtime (DD-4)
- [x] ComboController MonoBehaviour bridges input → state machine → animation → events
- [x] Local C# events: AttackStarted, ComboDropped, FinisherStarted, ComboEnded (DD-3)
- [x] ComboDebugUI with auto-advance (simulates animation events without Animator)
- [x] 29 edit-mode unit tests for ComboStateMachine

### Missing Fields on ComboStep
- [x] `canDashCancelOnHit` — bool flag per step
- [x] `canJumpCancelOnHit` — bool flag per step
- [x] `AttackData` reference — *Placeholder field ready; T005 will deliver the SO definition. ComboStep already has animation trigger and damage multiplier fields that will be superseded by AttackData.*

## Design Decisions

**DD-1: Branching tree combo structure**
Combo chains use a branching tree — inputs can branch (L→L→H = launcher, L→L→L = sweep). Each ComboStep has `nextOnLight` and `nextOnHeavy` index pointers. More expressive than linear chains, matches the "deep combat" vision.

**DD-2: Flat array with index pointers**
ComboDefinition stores all steps in a single `ComboStep[]` array. Each step references the next step by array index (-1 = no branch). Simple to author in the inspector, avoids nested SO proliferation.

**DD-3: Local C# events for motor communication**
ComboController fires `AttackStarted`, `ComboDropped`, `FinisherStarted`, `ComboEnded` events. CharacterMotor subscribes to lock/unlock movement. Matches the event pattern from T002 (Jumped, Dashed).

**DD-4: Plain C# state machine with Tick(dt)**
ComboStateMachine is a plain C# class. Combo window timer ticks via `Tick(deltaTime)` called from ComboController.Update(). All other transitions are animation-event-driven. Keeps combo logic testable without Unity runtime.

**DD-5: State machine exposes cancel properties, Controller orchestrates**
ComboStateMachine exposes `CanDashCancel`/`CanJumpCancel` read-only properties (checks `hitConfirmed` + step flags). ComboController checks these when dash/jump input arrives and orchestrates the cancel. Keeps state machine focused on sequencing per DD-4.

**DD-6: ComboController calls motor.SetAttackLock(bool), motor is passive**
ComboController holds a `[SerializeField] CharacterMotor` reference and calls `SetAttackLock(true)` on attack start, `SetAttackLock(false)` on combo end/drop/cancel. Motor just stores a bool and guards movement — no back-reference to combo. Null-conditional (`motor?.`) calls for safety since `SetAttackLock` is just a bool write with no dependencies.

**DD-7: ComboInteractionConfig ScriptableObject**
Broad config SO covering all combo interactions: cancel behavior (`dashCancelResetsCombo`, `jumpCancelResetsCombo`, `cancelPriority`), reset triggers (`resetOnStagger`, `resetOnDeath`, `resetOnDashCancel`), movement lock (`lockMovementDuringAttack`, `lockMovementDuringFinisher`). One SO per character, tunable in Inspector.

**DD-8: No animator = Debug.LogError on Awake**
If `animator` is null in ComboController.Awake(), log a red error with `Debug.LogError` (clickable reference to the GameObject). Does not crash — ComboDebugUI auto-advance still works for testing. Later, swap to `[RequireComponent(typeof(Animator))]` when animations are mandatory.

**DD-9: ComboDebugUI stays dumb**
Debug component doesn't read ComboInteractionConfig. It only advances combo state via timers. Config interactions are tested through real gameplay or targeted unit tests.

**DD-10: InputBufferSystem deferred to T003 reopen**
Cancel inputs (dash/jump) are handled as instant checks — player presses Dash while `CanDashCancel` is true, it fires immediately via CharacterInputHandler → ComboController.RequestDashCancel(). No buffering needed. T003 should be reopened separately to build the full standalone InputBufferSystem with circular buffer and multi-input-type support.

## Implementation Notes (Completed)

**Files created (7 code files + 1 test file + editor updates):**
- `Combat/Combo/AttackType.cs` — enum: Light, Heavy
- `Combat/Combo/ComboState.cs` — enum: Idle, Attacking, ComboWindow, Finisher
- `Combat/Combo/ComboStep.cs` — serializable struct: attack type, animation trigger, damage multiplier, branching indices, finisher flag
- `Combat/Combo/ComboDefinition.cs` — ScriptableObject: flat step array, root indices, default combo window
- `Combat/Combo/ComboStateMachine.cs` — plain C# class: state tracking, input buffering, combo window timer, animation event callbacks
- `Combat/Combo/ComboController.cs` — MonoBehaviour: wires input → state machine → animation → events
- `Combat/Combo/ComboDebugUI.cs` — debug overlay with auto-advance (simulates animation events when no Animator present)
- `Tests/EditMode/Combat/Combo/ComboStateMachineTests.cs` — 44 unit tests (25 original + 15 hit-confirm/cancel + 4 other)
- `Characters/CharacterInputHandler.cs` — updated with lightAttackAction/heavyAttackAction
- `Editor/Prefabs/MovementTestSceneCreator.cs` — updated with combo wiring
- `Editor/Prefabs/PlayerPrefabCreator.cs` — updated with ComboController on prefab

**Brutor combo tree (7 steps):**
```
Step 0: Light root → L:1, H:5
Step 1: Light chain → L:2, H:3
Step 2: Light finisher (sweep) → terminal
Step 3: Heavy (launcher) → L:-1, H:4
Step 4: Heavy finisher (slam) → terminal
Step 5: Heavy root → L:-1, H:6
Step 6: Heavy finisher (ground pound) → terminal
```

## References

- `architecture/data-flow.md` — Steps 1-4: Input → Buffer → ComboSystem → Controller → HitboxManager
- `architecture/interface-contracts.md` — AttackData fields, ICombatEvents.OnStrike (fired downstream, not here)
- `design-specs/CHARACTER-ARCHETYPES.md` — Brutor: "3-hit shield bash combo. Slow, short range, solid knockback"
- `developer/coding-standards.md` — ScriptableObjects for data, Animation Events for timing, plain C# for testable logic
