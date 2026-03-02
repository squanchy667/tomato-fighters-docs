# T012: CameraController2D

## Metadata
| Field | Value |
|-------|-------|
| **Phase** | 1 — Foundation |
| **Type** | implementation |
| **Priority** | P0 |
| **Owner** | Dev 3 |
| **Agent** | world-agent |
| **Depends On** | T001 |
| **Blocks** | T013, T051 |
| **Status** | PENDING |
| **Branch** | `pillar3/T012-camera-controller` |

## Objective
Implement a smooth side-scrolling camera that follows the player with configurable leading, respects level bounds set by WaveManager, and zooms in on stun events via SO event channels — all without directly referencing any Combat pillar code.

## Context
`CameraController2D` is the World pillar's camera system at `Assets/Scripts/World/CameraController2D.cs`. It supports the beat 'em up side-scrolling format: the camera follows the player horizontally with smooth damping and configurable leading (camera offset in the direction the player faces).

Key integrations:
- **WaveManager (T010)**: When a LevelBound is reached, WaveManager signals the camera to stop scrolling until the wave is cleared. This communication happens through a shared SO event channel or bool variable — NOT a direct reference
- **PressureSystem (T026, Phase 3)**: When an enemy is stunned, the camera zooms in for dramatic effect. The camera listens to a stun event SO channel — it does NOT reference PressureSystem directly
- **Co-op (T051, Phase 5)**: Future support for framing 2 players — design the follow target system to accept multiple targets

Architecture rules:
- No singletons
- SO-based event channels for cross-pillar communication
- All values configurable via `[SerializeField]` in Inspector

## Requirements
1. Create `CameraController2D` MonoBehaviour at `Assets/Scripts/World/CameraController2D.cs`
2. Smooth follow: camera follows a target transform with `Vector3.SmoothDamp` (configurable smooth time)
3. Configurable leading: offset the camera in the direction the player is facing (configurable lead distance and lead speed)
4. Level bounds:
   - Define min/max X (and optionally Y) bounds via `[SerializeField]`
   - Camera position is clamped to these bounds after follow calculation
   - Provide a public method or SO event listener to update bounds dynamically (WaveManager sets bounds per area)
5. Bound lock mode: when WaveManager signals a LevelBound trigger, camera stops following and locks at the bound position until wave is cleared
   - Listen to an SO event channel (e.g., `BoolEventChannel` or `VoidEventChannel`) for lock/unlock
   - Do NOT take a direct reference to WaveManager
6. Stun zoom: listen to a stun event SO channel
   - On stun: smoothly zoom in (reduce orthographic size) over configurable duration
   - On stun end: smoothly zoom back to default orthographic size
   - Zoom amount and duration configurable in Inspector
7. Co-op framing (future-proof):
   - Use a `List<Transform>` for follow targets instead of a single transform
   - When multiple targets: camera centers between them, adjusts orthographic size to frame both
   - For Phase 1: list has 1 entry (single player)
8. All camera updates in `LateUpdate()` — never `Update()` or `FixedUpdate()` for camera
9. Expose all tuning values via `[SerializeField]`:
   - Smooth time (follow damping)
   - Lead distance and lead speed
   - Default orthographic size
   - Stun zoom orthographic size
   - Zoom transition duration
   - Vertical offset (look-ahead Y)
   - Bounds min/max

## File Plan
| File | Purpose |
|------|---------|
| `Assets/Scripts/World/CameraController2D.cs` | Main camera follow, bounds, and zoom MonoBehaviour |

## Implementation Notes
- Use `Vector3.SmoothDamp` for follow — it handles acceleration/deceleration naturally and avoids jitter
- Leading: maintain a `desiredLead` that lerps toward the facing direction * lead distance. Add this to the target position before applying SmoothDamp
- Bounds clamping: after calculating the desired position (follow + lead + offset), clamp X and Y to the configured bounds. Use `Mathf.Clamp`
- Bound lock: use a bool flag `isLocked`. When locked, the camera target becomes the lock position (the LevelBound X), not the player. Smooth transition into and out of lock
- Stun zoom: use `Mathf.Lerp` or DOTween (if available) to transition `Camera.main.orthographicSize`. Keep it simple — a coroutine that lerps over the configured duration
- Do NOT use `Camera.main` repeatedly — cache the `Camera` component in `Awake()`
- Co-op framing formula: `center = average of all target positions`, `size = max distance between targets * 0.5f + padding`. Clamp size to min/max orthographic size
- For SO event listening: use `[SerializeField]` references to event channel SOs. Subscribe in `OnEnable`, unsubscribe in `OnDisable`
- The camera should work without WaveManager present (no null reference errors) — check for null/missing event channels gracefully

## Acceptance Criteria
- [ ] Smooth follow with configurable damping — no jitter or snapping
- [ ] Configurable leading in player's facing direction
- [ ] Level bounds respected — camera never shows beyond min/max
- [ ] Bound lock mode works: camera stops following when locked, resumes when unlocked
- [ ] Stun zoom: smoothly zooms in on stun event, zooms back on stun end
- [ ] Stun event listened to via SO event channel — no direct reference to PressureSystem
- [ ] Co-op framing structure in place (List<Transform> targets) — works with 1 player
- [ ] All values configurable in Inspector via `[SerializeField]`
- [ ] Camera updates in `LateUpdate()` only
- [ ] No references to Combat or Roguelite pillar code
- [ ] Compiles with zero warnings

## References
- [System Overview](../../architecture/system-overview.md) — World pillar camera system
- [Data Flow](../../architecture/data-flow.md) — Run Loop Flow, camera integrates with WaveManager bounds
- [TASK_BOARD.md](../../TASK_BOARD.md) — T012 entry
- [PLAN.md](../../PLAN.md) — Phase 1 scope, SO-based event channels
