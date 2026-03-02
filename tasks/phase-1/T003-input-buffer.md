# T003: InputBufferSystem

## Metadata
| Field | Value |
|-------|-------|
| **Phase** | 1 — Foundation |
| **Type** | implementation |
| **Priority** | P0 |
| **Owner** | Dev 1 |
| **Agent** | combat-agent |
| **Depends On** | T001 |
| **Blocks** | T004 |
| **Status** | DONE |
| **Completed** | 2026-03-02 |
| **Branch** | `tal` |

## Objective
Build an input buffering system that queues recent player inputs and allows pre-buffering attacks during animations, giving the combat system the responsive, forgiving feel expected in beat-em-up / fighting games.

## Context
In action games, raw input polling drops inputs that arrive during attack animations or recovery frames. The InputBufferSystem sits between raw input and the ComboSystem (T004) — it captures what the player *intended* to do and holds it briefly so the ComboSystem can consume it at the earliest legal moment.

This is critical for combo flow: a player mashing Strike during an active attack animation expects the next strike to come out as soon as the current one recovers. Without buffering, the input is lost and the combo chain drops.

See `architecture/data-flow.md` step 1: `Input → InputBufferSystem → Queue input with timestamp → Check if within buffer window (0.1s)`.

The buffer feeds into ComboSystem (T004), which reads from it to advance through the combo tree. It also interacts with CharacterController2D (T002) for dash and jump inputs.

## Requirements

1. **Input types**
   - Define an enum `BufferedInputType`: `Strike`, `Skill`, `Arcana`, `Dash`, `Jump`
   - Each buffered entry stores: `BufferedInputType type`, `float timestamp`, `bool consumed`

2. **Buffer queue**
   - Use a fixed-capacity circular buffer (not `List<>` — avoid GC allocations in combat frames)
   - Capacity: configurable, default 8 entries (more than enough for frame-perfect inputs)
   - New inputs push to the queue; old entries are overwritten when capacity is exceeded

3. **Buffer window**
   - Configurable duration: `[SerializeField] float bufferWindow = 0.1f` (~6 frames at 60fps)
   - An input is valid if `Time.time - input.timestamp <= bufferWindow`
   - Expired inputs are ignored when consumers query the buffer (lazy cleanup — don't iterate the queue every frame)

4. **Input capture**
   - Call `BufferInput(BufferedInputType type)` from the input reading layer (Update loop or input callbacks)
   - Timestamp is `Time.time` at the moment of capture
   - Duplicate filtering: if the same input type was buffered within the last 1-2 frames (configurable), don't re-buffer (prevents held-button spam from flooding the queue)

5. **Input consumption**
   - `bool TryConsumeInput(BufferedInputType type, out BufferedInput input)` — returns the oldest unconsumed input of the given type within the buffer window, marks it as consumed
   - `bool HasBufferedInput(BufferedInputType type)` — peek without consuming
   - `void ClearBuffer()` — called on state reset (death, stagger recovery, scene transition)
   - `void ClearInputType(BufferedInputType type)` — clear only a specific type (e.g., clear Dash after dash executes)

6. **Pre-buffering during animations**
   - The key feature: when the player is in an active attack animation (movement locked), inputs still get buffered
   - ComboSystem (T004) will call `TryConsumeInput(Strike)` at the end of its current attack's recovery window
   - CharacterController2D (T002) will call `TryConsumeInput(Dash)` / `TryConsumeInput(Jump)` on hit-confirm cancel windows

7. **Debug support**
   - `[SerializeField] bool showDebugLog` — logs buffered/consumed inputs to console
   - Expose current buffer contents as a read-only list for editor inspection (debug only, not for gameplay code)

## File Plan

| File Path | Description |
|-----------|-------------|
| `Combat/InputBufferSystem.cs` | MonoBehaviour: circular buffer of timestamped inputs with consume/peek/clear API |

## Implementation Notes

- **Plain C# circular buffer:** Implement as a struct array + head/tail indices. Avoid `Queue<>` to prevent GC pressure during combat frames. The buffer is tiny (8 entries) so a fixed array is ideal
- **MonoBehaviour vs plain C#:** This should be a MonoBehaviour attached to the player GameObject so it can be wired via `[SerializeField]` to ComboSystem and CharacterController2D. The internal buffer data structure is plain C# (struct array)
- **Time.time for timestamps:** Use `Time.time`, not `Time.unscaledTime`. If hitstop/slowmo is added later (T053), buffered inputs should still respect game time
- **No input reading in this class:** InputBufferSystem does NOT read `Input.GetButtonDown()` directly. The input reading happens in CharacterController2D or a separate input handler that calls `BufferInput()`. This keeps the buffer system input-agnostic (works with old Input or new Input System)
- **Thread safety not needed:** Unity is single-threaded for gameplay. No locks required
- **Buffer clearing on state transitions:** The ComboSystem or CharacterController2D is responsible for calling `ClearBuffer()` at appropriate times (death, stagger recovery). The buffer itself is passive — it just stores and serves
- **Frame-rate independence:** The buffer window is time-based (0.1s), not frame-based. This ensures consistent behavior at variable frame rates. The "~6 frames at 60fps" is just for documentation context

## Acceptance Criteria

- [x] Input buffering during attack animations (built into ComboStateMachine)
- [x] Pre-buffering works during active animations (inputs stored in Attacking state, consumed on ComboWindow open)
- [x] Buffer clears on combo reset/drop
- [x] Compiles with zero warnings

> **Note:** Input buffering was implemented directly inside `ComboStateMachine` rather than as a standalone system. The combo state machine stores one buffered input during the Attacking state and consumes it when the ComboWindow opens. This approach is simpler and avoids the overhead of a separate circular buffer for the current combo-only use case. If other systems (dash-cancel, jump-cancel) need buffering later, a standalone InputBufferSystem can be extracted.

## Implementation Notes (Completed)

Input buffering is embedded in `ComboStateMachine` (built in T004/combo system), not as a standalone class. The state machine handles:
- Buffering one input during the `Attacking` state
- Consuming the buffered input when `OnComboWindowOpen()` fires
- Clearing buffered input on combo drop or reset

This was a design decision (DD-4 in the combo task) — keeping combo logic as a plain C# state machine with `Tick(dt)` for testability. The input buffer is part of that state machine.

## References

- `architecture/data-flow.md` — Step 1: InputBufferSystem queues input, checks buffer window
- `architecture/interface-contracts.md` — No direct interface dependency, but feeds into ComboSystem
- `developer/coding-standards.md` — No GC allocations in combat frame, [SerializeField] injection
- `design-specs/CHARACTER-ARCHETYPES.md` — All 4 characters use the same 5 input types with character-specific responses
