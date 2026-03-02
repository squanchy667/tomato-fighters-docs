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
| **Completed** | 2026-03-02 |
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

| File Path | Description |
|-----------|-------------|
| `Combat/ComboNode.cs` | Serializable class or ScriptableObject defining a single node in the combo tree |
| `Combat/ComboSystem.cs` | MonoBehaviour: combo state machine, InputBuffer consumer, attack flow controller |

## Implementation Notes

- **ComboNode as ScriptableObject vs plain class:** If using ScriptableObject, each node is an asset and branches are wired via Inspector drag-and-drop. This is clean but can be tedious for large trees. Alternative: a single `ComboTreeData` ScriptableObject that holds a serialized list of nodes with index-based branching. Either approach works — pick whichever is easier to author and extend to 4 characters in T014
- **No damage calculation here:** ComboSystem only manages the attack flow (which attack, when, cancel windows). Actual damage calculation happens in the Damage Calculation step (see `architecture/data-flow.md` step 5) which involves HitboxManager + IBuffProvider. ComboSystem just loads the AttackData and makes it available
- **Animator integration:** For Phase 1, the ComboSystem triggers animations via `Animator.Play("Strike1")` or `Animator.SetTrigger("strike")`. The exact mechanism depends on T024 (Character Animator Controllers). For now, expose `OnAttackStarted(AttackData)` as an event that any animation system can subscribe to
- **Cancel priority:** If both dash-cancel and jump-cancel are available and both inputs are buffered, dash takes priority (more common in beat-em-ups). This is configurable
- **Combo reset:** Combo resets (returns to Idle) on: input window expiry, taking damage (stagger), death, dash-cancel execution, manual reset call. It does NOT reset on jump-cancel (player can attack from air in future)
- **No ICombatEvents firing yet:** The ICombatEvents.OnStrike event will be fired by the damage pipeline, not directly by ComboSystem. ComboSystem is purely about attack sequencing

## Acceptance Criteria

- [x] Branching combo tree with light/heavy paths per step (ComboStep struct with nextOnLight/nextOnHeavy indices)
- [x] ComboStep supports branching to different nodes based on AttackType (Light/Heavy)
- [x] ComboStateMachine: Idle → Attacking → ComboWindow → Finisher (4-state machine)
- [x] Brutor's 7-step branching tree: L→L→L (sweep), L→H→H (launcher→slam), L→L→H (slam), H→H (ground pound)
- [x] Each step has configurable combo window duration (per-step override or definition default)
- [x] Input buffering during attack animations (built into ComboStateMachine)
- [x] Animation-event-driven transitions (OnComboWindowOpen, OnFinisherEnd)
- [x] Finisher detection when chain reaches terminal step
- [x] Local C# events: AttackStarted, ComboDropped, FinisherStarted, ComboEnded
- [x] ComboDefinition ScriptableObject with flat step array (DD-2)
- [x] Plain C# ComboStateMachine testable without Unity runtime (DD-4)
- [x] ComboController MonoBehaviour bridges input → state machine → animation → events
- [x] 25 edit-mode unit tests for ComboStateMachine
- [x] Compiles with zero warnings

## Design Decisions

**DD-1: Branching tree combo structure**
Combo chains use a branching tree — inputs can branch (L→L→H = launcher, L→L→L = sweep). Each ComboStep has `nextOnLight` and `nextOnHeavy` index pointers. More expressive than linear chains, matches the "deep combat" vision.

**DD-2: Flat array with index pointers**
ComboDefinition stores all steps in a single `ComboStep[]` array. Each step references the next step by array index (-1 = no branch). Simple to author in the inspector, avoids nested SO proliferation.

**DD-3: Local C# events for motor communication**
ComboController fires `AttackStarted`, `ComboDropped`, `FinisherStarted`, `ComboEnded` events. CharacterMotor subscribes to lock/unlock movement. Matches the event pattern from T002 (Jumped, Dashed).

**DD-4: Plain C# state machine with Tick(dt)**
ComboStateMachine is a plain C# class. Combo window timer ticks via `Tick(deltaTime)` called from ComboController.Update(). All other transitions are animation-event-driven. Keeps combo logic testable without Unity runtime.

## Implementation Notes (Completed)

**Files created (7 code files + 1 test file + editor updates):**
- `Combat/Combo/AttackType.cs` — enum: Light, Heavy
- `Combat/Combo/ComboState.cs` — enum: Idle, Attacking, ComboWindow, Finisher
- `Combat/Combo/ComboStep.cs` — serializable struct: attack type, animation trigger, damage multiplier, branching indices, finisher flag
- `Combat/Combo/ComboDefinition.cs` — ScriptableObject: flat step array, root indices, default combo window
- `Combat/Combo/ComboStateMachine.cs` — plain C# class: state tracking, input buffering, combo window timer, animation event callbacks
- `Combat/Combo/ComboController.cs` — MonoBehaviour: wires input → state machine → animation → events
- `Combat/Combo/ComboDebugUI.cs` — debug overlay with auto-advance (simulates animation events when no Animator present)
- `Tests/EditMode/Combat/Combo/ComboStateMachineTests.cs` — 25 unit tests
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
