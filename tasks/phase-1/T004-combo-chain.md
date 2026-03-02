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
| **Status** | PENDING |
| **Branch** | `pillar1/T004-combo-chain` |

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

- [ ] ComboNode class with AttackData reference, input branches, input window, and cancel flags
- [ ] ComboNode supports branching to different nodes based on BufferedInputType
- [ ] ComboSystem state machine: Idle → Attacking → WindowOpen → Recovering → Idle
- [ ] Brutor's 4-node combo tree: Shield Jab → Shield Swipe → Shield Bash → Overhead Slam
- [ ] Each node has configurable input window duration
- [ ] Hit-confirm callback (`OnHitConfirmed`) enables dash-cancel and jump-cancel flags
- [ ] Dash-cancel on hit-confirm: consumes buffered Dash, cancels recovery, triggers dash
- [ ] Jump-cancel on hit-confirm: consumes buffered Jump, cancels recovery, triggers jump
- [ ] InputBufferSystem integration: consumes buffered inputs for combo branching
- [ ] CharacterController2D integration: movement locked during attacks, unlocked on combo end
- [ ] Animation-driven timing via AttackData frame data (with timer fallback until Animation Events exist)
- [ ] Combo resets on: window expiry, stagger, death, dash-cancel
- [ ] `isFinisher` flag on final combo node
- [ ] Compiles with zero warnings

## References

- `architecture/data-flow.md` — Steps 1-4: Input → Buffer → ComboSystem → Controller → HitboxManager
- `architecture/interface-contracts.md` — AttackData fields, ICombatEvents.OnStrike (fired downstream, not here)
- `design-specs/CHARACTER-ARCHETYPES.md` — Brutor: "3-hit shield bash combo. Slow, short range, solid knockback"
- `developer/coding-standards.md` — ScriptableObjects for data, Animation Events for timing, plain C# for testable logic
