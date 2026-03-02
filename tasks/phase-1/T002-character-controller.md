# T002: CharacterController2D — Brutor First

## Metadata
| Field | Value |
|-------|-------|
| **Phase** | 1 — Foundation |
| **Type** | implementation |
| **Priority** | P0 |
| **Owner** | Dev 1 |
| **Agent** | combat-agent |
| **Depends On** | T001 |
| **Blocks** | T004, T013 |
| **Status** | DONE |
| **Completed** | 2026-03-02 |
| **Branch** | `tal` |

## Objective
Build the core 2D character controller that handles 8-directional movement, jumping with gravity, and forward dashing — initially tuned for Brutor's heavy, tanky feel. This controller is the physical foundation that every combat action builds on top of.

## Context
This is the first Combat-pillar MonoBehaviour. It sits at the top of the per-frame input pipeline (see `architecture/data-flow.md`): player input enters here, gets translated to Rigidbody2D forces, and feeds into the combo/attack systems downstream.

Brutor is the first character implemented because his movement is the simplest (slowest speed, shortest dash, armored dash with super-armor). Once this works for Brutor, it will be parameterized for the other 3 characters (Slasher's fast dash-through, Mystica's blink teleport, Viper's backflip) — but that parameterization happens in later tasks.

Key design constraints from `developer/coding-standards.md`:
- **Rigidbody2D for ALL physics** — movement, jump, dash, knockback. Never `transform.position`.
- **No singletons** — dependencies injected via `[SerializeField]`.
- **All values in Inspector** — speed, jump force, dash distance, gravity scale, everything.
- **Interface-only coupling** — reads speed multiplier from `IBuffProvider`/`IPathProvider`, but never imports from Roguelite namespace.

Brutor's base stats: SPD 0.7 (lowest of 4 characters). His dash is a short armored charge with super-armor during startup (can't be interrupted, still takes damage).

## Requirements

1. **8-directional movement**
   - Read horizontal and vertical input (Unity Input System or `Input.GetAxis`)
   - Normalize diagonal movement to prevent faster diagonal speed
   - Apply movement via `Rigidbody2D.velocity` (horizontal) — never `transform.Translate`
   - Base move speed configurable via `[SerializeField] float moveSpeed`
   - Multiply by `IPathProvider.GetPathStatBonus(StatType.Speed)` when available (default 1.0)
   - Flip sprite direction based on movement (SpriteRenderer or transform.localScale.x)

2. **Jump with gravity**
   - Single jump (no double jump at baseline — could be added via path ability)
   - Apply vertical impulse via `Rigidbody2D.AddForce(Vector2.up * jumpForce, ForceMode2D.Impulse)`
   - Configurable: `jumpForce`, `gravityScale`, `fallGravityMultiplier` (heavier fall for game feel)
   - Ground check via small `OverlapCircle` or `BoxCast` at feet — configurable `groundCheckRadius` and `groundLayer`
   - Coyote time: brief window after leaving platform where jump still registers (configurable, ~0.1s)
   - Jump buffering: if jump pressed just before landing, execute on land (coordinate with InputBufferSystem in T003)

3. **Forward dash**
   - Dash in facing direction (or input direction)
   - Apply dash as velocity override: `Rigidbody2D.velocity = dashDirection * dashSpeed` for `dashDuration` seconds
   - Configurable: `dashSpeed`, `dashDuration`, `dashCooldown`
   - Dash state flag: `isDashing` — other systems read this for i-frame timing
   - **Brutor-specific: Armored Dash** — during dash startup frames, set `hasSuperArmor = true`. Super-armor means the character cannot be staggered/interrupted but still takes damage. The flag is read by DefenseSystem (T016) later
   - Dash locks movement input for its duration

4. **State management**
   - Track current state: `Idle`, `Moving`, `Jumping`, `Falling`, `Dashing`, `Attacking` (attacking state set externally by ComboSystem)
   - Movement locked during: dash, attack, stagger (stagger set externally)
   - Expose `bool CanMove`, `bool CanDash`, `bool CanJump` for other systems to query
   - Expose `bool IsGrounded`, `bool IsDashing`, `bool HasSuperArmor` as public read-only properties

5. **Integration hooks**
   - `[SerializeField] private Rigidbody2D rb` — required component
   - `[SerializeField] private IBuffProvider buffProvider` — optional, returns 1.0x if null (use a serialized MonoBehaviour that implements the interface, or wire at runtime)
   - `OnDashStart` and `OnDashEnd` callback events for other systems (DefenseSystem will hook i-frames here)
   - `SetMovementLock(bool locked)` — called by ComboSystem during attacks, by DefenseSystem during stagger

## File Plan

| File Path | Description |
|-----------|-------------|
| `Combat/CharacterController2D.cs` | MonoBehaviour: movement, jump, dash, state tracking for player character |

## Implementation Notes

- **Rigidbody2D settings:** Gravity Scale should default to the controller's configured value. Use `rb.gravityScale` dynamically — increase during fall for heavier feel (`fallGravityMultiplier`), reset on jump
- **Dash implementation:** Override velocity directly during dash, then restore normal movement control after dash duration expires. Use a coroutine or timer, not `Invoke`
- **Super-armor flag:** This is just a bool property. DefenseSystem (T016) will check it when processing incoming hits. If `hasSuperArmor`, skip stagger/knockback but still apply damage. Set true during Brutor's dash startup frames, false after
- **Facing direction:** Store as `int facingDirection` (+1 or -1). Flip via `transform.localScale = new Vector3(facingDirection, 1, 1)`. Do NOT flip during dash — dash uses the facing direction at dash start
- **No Input System dependency yet:** Use `Input.GetAxisRaw("Horizontal")` / `Input.GetAxisRaw("Vertical")` for now. Unity's new Input System integration can be added later without changing the controller's internal logic — just swap the input source
- **Inspector organization:** Use `[Header("Movement")]`, `[Header("Jump")]`, `[Header("Dash")]`, `[Header("Debug")]` groups. Use `[Tooltip]` on non-obvious fields. Use `[Range]` on bounded values
- **Null-safe IBuffProvider:** In `Awake()`, if `buffProvider` is not assigned, log a warning and use a fallback that returns 1.0 multipliers. Combat frame code must never throw

## Acceptance Criteria

- [x] 8-directional movement with configurable speed via `[SerializeField]`
- [x] Diagonal movement normalized (no faster diagonal speed)
- [x] Jump with configurable gravity and fall multiplier
- [x] Ground check — belt-scroll model: grounded = `jumpHeight <= 0` (no collider needed)
- [x] Coyote time window (configurable duration)
- [x] Dash with configurable speed, duration, and cooldown
- [x] Dash locks movement input for its duration
- [x] Brutor's armored dash: `hasSuperArmor` flag active during dash startup
- [x] All physics through Rigidbody2D — zero `transform.position` writes
- [x] All values exposed in Inspector via `[SerializeField]` with `[Header]` groups
- [x] Sprite facing flips based on movement direction
- [x] Public read-only properties: `IsGrounded`, `IsDashing`, `HasSuperArmor`, `CanMove`, `CanDash`, `CanJump`
- [x] `SetMovementLock` method for external systems
- [x] Dash start/end events for other systems to hook into
- [x] Null-safe `IBuffProvider` access (falls back to 1.0x multipliers)
- [x] Compiles with zero warnings

## Implementation Notes (Completed)

**Belt-scroll rework:** The original spec described a platformer-style controller with gravity. During implementation, this was reworked to a belt-scroll model (Streets of Rage style):
- X = horizontal movement, Y = depth movement (walk into/out of screen)
- Jump = sprite offset (visual only), collider stays on ground plane
- `Rigidbody2D.gravityScale = 0` always; jump arc simulated with manual `jumpVelocity` / `jumpGravity`
- No ground collider needed — grounded is purely `jumpHeight <= 0`
- Shadow child stays at feet while sprite child offsets by jumpHeight

**Files created (6 code files + 3 editor/test files):**
- `Characters/CharacterMotor.cs` — MonoBehaviour: belt-scroll movement, jump, dash with Rigidbody2D
- `Characters/MovementStateMachine.cs` — Plain C# state machine (Grounded/Airborne/Dashing)
- `Characters/MovementConfig.cs` — ScriptableObject with all tuning params
- `Characters/CharacterInputHandler.cs` — Unity new Input System (InputActionReference)
- `Characters/MovementState.cs` — 3-state enum
- `Editor/Prefabs/PlayerPrefabCreator.cs` — Editor menu script for player prefab
- `Editor/Prefabs/MovementTestSceneCreator.cs` — Editor menu script for test scene
- `Tests/EditMode/Characters/MovementStateMachineTests.cs` — 17 unit tests

**Design decisions:**
- Belt-scroll movement model (no gravity, XY ground plane, simulated jump)
- Plain C# MovementStateMachine with Tick(dt) for testability
- GroundDetector deleted — grounded is purely `jumpHeight <= 0`
- Default prefab character = Brutor (SPD 0.7, tankiest)

## References

- `architecture/data-flow.md` — Input → Controller → ComboSystem → HitboxManager pipeline
- `architecture/interface-contracts.md` — IBuffProvider, IPathProvider method signatures
- `design-specs/CHARACTER-ARCHETYPES.md` — Brutor: SPD 0.7, "Short armored charge. Has super-armor during startup"
- `developer/coding-standards.md` — Rigidbody2D for physics, [SerializeField] injection, [Header]/[Tooltip] conventions
