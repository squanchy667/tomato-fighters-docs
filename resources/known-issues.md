# Known Issues

*(No active bugs. 12 tasks completed across Phase 1–2.)*

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
