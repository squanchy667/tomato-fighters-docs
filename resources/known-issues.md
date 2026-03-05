# Known Issues

## Phase 1 Demo Playtest (2026-03-05)

> MUST VALIDATE ALL P0/P1 RESOLVED BEFORE PHASE 2 CLOSES

### P0 — Critical (Blocks Demo)

| # | Issue | Owner | Component | Details |
|---|-------|-------|-----------|---------|
| 1 | Player death not handled | Dev 1 | `PlayerDamageable` | No death state — player keeps moving at 0 HP. Need: death state, disable input, death animation, respawn/game-over. |
| 2 | Enemy AI can't find player | Dev 3 | `EnemyAI` | `playerLayer mask=512` (layer 9 = PlayerHurtbox) doesn't match spawned player layer. `BasicEnemyPrefabCreator` logs `Layer 'Player' not found`. Align player layer with EnemyAI scan mask. |
| 3 | Balance: player too weak | Dev 1 | `AttackData` SOs, `CharacterBaseStats` | Player damage output too low vs enemy HP (80). Brutor ATK=0.7, enemy BasicSlash=0.6x. Tune damage multipliers + base ATK values. |

### P1 — High (Quality of Life)

| # | Issue | Owner | Component | Details |
|---|-------|-------|-----------|---------|
| 4 | Camera doesn't follow spawned player | Dev 3 | `CameraController2D` | Target stays on SpawnPoint after CharacterSpawner instantiates. Spawner must notify camera of new target. |
| 5 | No respawn/restart after death | Dev 3 | New | No way to restart after player dies. Need: reload scene key (R) or respawn timer. |
| 6 | Enemies active during character select | Dev 3 | `EnemyAI` | `Time.timeScale=0` pauses physics but `Update()` still fires. Check timeScale or start dormant until game begins. |
| 7 | Enemy HP shows 0/80 on spawn | Dev 1 | `EnemyBase`, `DebugHealthBar` | Script execution order: `DebugHealthBar.Awake()` reads HP before `EnemyBase.Awake()` sets it. |

### P2 — Polish

| # | Issue | Owner | Component | Details |
|---|-------|-------|-----------|---------|
| 8 | No hit feedback | Dev 1 | Combat system | No hitstop, screen shake, or flash on hit. Makes combat feel flat. Basic version before T053. |
| 9 | No death animation for enemies | Dev 3 | `EnemyBase` | Just `Destroy(gameObject, 1f)` — no fall, no fade. |
| 10 | HUD not wired to spawned player | Dev 3 | `HUDManager` | SO event refs lost during prefab serialization. Needs Resources fallback like CharacterInputHandler. |

### Recommended Fix Order
1. #2 Enemy AI targeting (Dev 3)
2. #3 Balance tuning (Dev 1)
3. #1 Player death (Dev 1)
4. #4 Camera follow (Dev 3)
5. #5 Restart (Dev 3)
6. Rest in priority order

---

## Resolved Issues (2026-03-04)

| # | Bug | Component | Root Cause | Fix |
|---|-----|-----------|-----------|-----|
| 1 | DebugHealthBar not rendering | `Shared/Components/DebugHealthBar.cs` | `Image.Type.Filled` doesn't render without a proper source sprite in Unity 2022 | Switched to anchor-based fill (`anchorMax.x = ratio`) with `Image.Type.Simple` |
| 2 | PlayerDamageable flash invisible | `Combat/PlayerDamageable.cs` | `GetComponent<SpriteRenderer>()` found root's invisible renderer (no sprite), not the Sprite child | Changed to `GetComponentInChildren<SpriteRenderer>()` |
| 3 | Trigger colliders not detecting overlaps on re-enable | `Shared/Components/HitboxDamage.cs` | `autoSyncTransforms=false` means re-enabled triggers don't fire `OnTriggerEnter2D` while already overlapping | Added `Physics2D.SyncTransforms()` + `Rigidbody2D.WakeUp()` in `OnEnable()` |

**Temporary diagnostic tooling still in codebase:**
- `DamagePipelineDiagnostic.cs` at `Assets/` root (outside asmdef) — validates full damage pipeline on Start
- 3 `Debug.Log` lines in `HitboxManager.cs` for pipeline tracing
- 1 `Debug.Log` in `HitboxDamage.OnTriggerStay2D`
- These should be removed or gated behind `#if UNITY_EDITOR` before release builds

## Expected Challenges

| Area | Risk | Mitigation |
|------|------|-----------|
| Path ability balance | 24 builds hard to balance | Dedicated balance pass in Phase 5-6 |
| Animation volume | 4 characters × many states | Animator Override Controllers, shared base |
| Co-op netcode | Fighting game precision needed | Evaluate Fishnet vs Mirror early, rollback essential |
| Cross-pillar integration | Interface mismatches | T001 sync session, weekly integration builds |
| VFX performance | 36 path ability effects | Particle budget per effect (50 max), pooling |
