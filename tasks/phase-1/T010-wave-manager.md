# T010: WaveManager

## Metadata
| Field | Value |
|-------|-------|
| **Phase** | 1 — Foundation |
| **Type** | implementation |
| **Priority** | P0 |
| **Owner** | Dev 3 |
| **Agent** | world-agent |
| **Depends On** | T001 |
| **Blocks** | T013, T033 |
| **Status** | DONE |
| **Branch** | `pillar3/T010-wave-manager` |

## Objective
Implement a configurable wave spawning system that controls enemy composition per wave, pauses camera scrolling at level bounds until waves are cleared, and fires events to drive the run loop (reward selection, area transitions).

## Context
WaveManager is a core World pillar system that drives the combat encounter pacing. It lives at `Assets/Scripts/World/WaveManager.cs`. The run loop depends on this: `WaveManager spawns → fight → clear → OnAreaCleared → RewardSelectorUI picks ritual`. Camera stops at LevelBound triggers until the wave is cleared, then resumes side-scrolling.

This system communicates downstream through SO-based event channels (defined in `Shared/Events/`). It must NOT reference Combat or Roguelite pillar code directly — only fire events that other systems subscribe to.

Key architecture rules:
- No singletons — wire dependencies via `[SerializeField]`
- ScriptableObjects for all data (wave definitions, enemy spawn configs)
- Rigidbody2D for physics (enemy spawns should respect physics)
- SO-based event channels for cross-pillar communication

## Requirements
1. Create `WaveManager` MonoBehaviour at `Assets/Scripts/World/WaveManager.cs`
2. Define `EnemySpawnData` serializable struct: enemy prefab reference, spawn count, spawn delay, spawn position offset
3. Define `WaveData` serializable class: list of `EnemySpawnData`, optional flag (for side paths), wave clear condition (all enemies defeated)
4. Expose a `[SerializeField] List<WaveData>` for Inspector-configurable wave composition per area
5. Spawn enemies according to `EnemySpawnData` — instantiate prefabs at designated spawn points with configurable delays (coroutine-based)
6. Track alive enemy count per wave — decrement on enemy death (subscribe to enemy death events or use `IDamageable` death callback)
7. Implement LevelBound integration: when player reaches a LevelBound trigger collider, notify `CameraController2D` to stop scrolling and begin the next wave
8. Fire `OnWaveStart` event (SO event channel) when a wave begins — include wave index and enemy count
9. Fire `OnWaveCleared` event when all enemies in current wave are defeated
10. Fire `OnAreaComplete` event when all waves in the current area are cleared — this triggers reward selection downstream
11. Support optional waves: waves marked as optional can be skipped (side path mechanic) — player chooses to engage or bypass
12. Provide a public method to advance to the next area (called after reward selection completes)
13. All timing values (spawn delay, wave transition delay) configurable via `[SerializeField]` in Inspector

## Design Decisions

### DD-1: SO Event Channels for ALL Wave Events
All three wave events (`OnWaveStart`, `OnWaveCleared`, `OnAreaComplete`) use SO-based event channels — not C# events. This keeps communication consistent, allows Inspector-wired subscriptions (Camera, HUD, future Roguelite systems), and avoids mixing patterns. Uses two reusable channel types:
- `VoidEventChannel` — parameterless (OnWaveCleared, OnAreaComplete)
- `IntEventChannel` — carries int payload (OnWaveStart with wave index)

### DD-2: Separate Files for Data Structs
`EnemySpawnData` and `WaveData` live in `Shared/Data/` as separate files, consistent with `DamagePacket.cs`, `AttackData.cs`. Other pillars (T033 Branching Paths) may reference `EnemySpawnData` later.

### DD-3: Subscribe to EnemyBase.OnDied for Death Tracking
WaveManager subscribes to `EnemyBase.OnDied` (existing `event Action` on EnemyBase) when spawning each enemy. Decrements alive counter, checks wave clear. Same-pillar reference — no cross-pillar violation.

### DD-4: LevelBound as Separate MonoBehaviour
`LevelBound.cs` is a standalone trigger collider placed in the scene. Holds a `[SerializeField]` reference to a `VoidEventChannel` SO that WaveManager subscribes to. Keeps level layout concerns separate from spawn logic. Reusable across areas.

### DD-5: Explicit Transform Spawn Points
Spawn positions use `Transform[] spawnPoints` serialized on WaveManager — visible in Scene view, flexible for level design. `EnemySpawnData` references a spawn point index rather than raw offset vectors.

## File Plan
| File | Purpose |
|------|---------|
| `Assets/Scripts/Shared/Data/EnemySpawnData.cs` | `[Serializable]` struct: enemy prefab ref, count, spawn delay, spawn point index |
| `Assets/Scripts/Shared/Data/WaveData.cs` | `[Serializable]` class: `List<EnemySpawnData>`, optional flag, wave name |
| `Assets/Scripts/Shared/Events/VoidEventChannel.cs` | Reusable SO event channel (parameterless). Used for OnWaveCleared, OnAreaComplete, camera lock |
| `Assets/Scripts/Shared/Events/IntEventChannel.cs` | Reusable SO event channel (int payload). Used for OnWaveStart(waveIndex) |
| `Assets/Scripts/World/LevelBound.cs` | Trigger collider MonoBehaviour: `OnTriggerEnter2D` → fires SO event channel |
| `Assets/Scripts/World/WaveManager.cs` | Main MonoBehaviour: wave sequencing, enemy spawning (coroutines), alive tracking, event firing |

## Implementation Notes
- Use `[System.Serializable]` for `EnemySpawnData` and `WaveData` so they appear in Inspector
- Spawn enemies via coroutine with configurable delay between spawns (staggered entrance)
- Track alive enemies using a counter — increment on spawn, decrement via `EnemyBase.OnDied` subscription per spawned enemy
- LevelBound fires a `VoidEventChannel` SO on player enter. WaveManager subscribes to that channel and starts the next wave
- Camera lock/unlock communicated via a separate `VoidEventChannel` pair (OnCameraLock / OnCameraUnlock) — CameraController2D (T012) will subscribe when built
- Optional waves: bool flag on `WaveData`. For Phase 1, optional waves auto-play (UI prompt deferred). Skip logic present but not wired to UI
- Area complete fires `OnAreaComplete` channel — Roguelite's `RewardSelectorUI` (T031) subscribes later. For now, just fire the event
- Simple `Instantiate`/`Destroy` for Phase 1. Object pooling deferred to optimization pass
- Draw spawn point gizmos in `OnDrawGizmos` for level design visibility

## Acceptance Criteria
- [ ] Wave list with configurable enemy spawns visible and editable in Inspector
- [ ] LevelBound trigger stops camera scrolling and initiates wave
- [ ] Events fire correctly: `OnWaveStart` (with wave index), `OnWaveCleared`, `OnAreaComplete`
- [ ] Optional waves can be marked and skipped without breaking progression
- [ ] Enemy spawn delay is staggered and configurable
- [ ] Alive enemy tracking correctly decrements on death
- [ ] All values (spawn delay, wave transition delay, spawn positions) configurable via `[SerializeField]`
- [ ] No references to Combat or Roguelite pillar code — only Shared interfaces and SO events
- [ ] Compiles with zero warnings

## References
- [System Overview](../../architecture/system-overview.md) — World pillar, WaveManager role
- [Data Flow](../../architecture/data-flow.md) — Run Loop Flow: WaveManager → spawn → fight → clear → OnAreaCleared
- [TASK_BOARD.md](../../TASK_BOARD.md) — T010 entry
- [PLAN.md](../../PLAN.md) — Phase 1 scope, three-pillar architecture
