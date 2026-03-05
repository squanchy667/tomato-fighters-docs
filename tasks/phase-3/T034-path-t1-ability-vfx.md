# T034: Path T1 Ability VFX

**Type:** implementation | **Priority:** P1 | **Owner:** Dev 3 | **Depends on:** T028
**Status:** PENDING

## Summary

Create 12 VFX prefabs (one per T1 path ability) via a Creator Script, plus a data-driven lookup SO. Wire VFX spawning into existing T1 ability code so abilities have visible feedback at runtime.

## Deliverables

- 1 Creator Script: `Editor/AbilityVfxCreator.cs`
- 1 ScriptableObject class: `Scripts/Shared/Data/AbilityVfxLookup.cs`
- 1 generated SO asset: `ScriptableObjects/VFX/AbilityVfxLookup.asset`
- 12 generated VFX prefabs: `Prefabs/Effects/Abilities/{AbilityId}_VFX.prefab`
- 12 modified T1 ability files (add VFX spawn calls)
- 1 modified file: `PathAbilityExecutor.cs` (inject VfxLookup reference, pass prefab to abilities)
- 1 modified file: `PathAbilityContext.cs` (add `VfxPrefab` field)

## Design Decisions

### DD-1: AbilityVfxLookup SO for VFX references
**Decision:** Separate `AbilityVfxLookup` ScriptableObject maps `abilityId → vfxPrefab`.
**Rationale:** Keeps VFX data out of PathData (Roguelite/Shared). VFX is a World-pillar concern. Scales cleanly to T2/T3 VFX. PathAbilityExecutor queries the lookup when creating abilities and passes the prefab via PathAbilityContext.

```csharp
[CreateAssetMenu(fileName = "AbilityVfxLookup", menuName = "TomatoFighters/VFX/AbilityVfxLookup")]
public class AbilityVfxLookup : ScriptableObject
{
    [System.Serializable]
    public struct Entry
    {
        public string abilityId;
        public GameObject vfxPrefab;
    }

    [SerializeField] private Entry[] entries;

    public GameObject GetVfx(string abilityId)
    {
        foreach (var e in entries)
            if (e.abilityId == abilityId) return e.vfxPrefab;
        return null;
    }
}
```

### DD-2: Fire-and-forget VFX lifetime
**Decision:** `Object.Instantiate()` + `Object.Destroy(go, duration)` for all burst effects.
**Rationale:** Max 50 particles per effect, only 2 abilities active simultaneously. GC pressure is negligible. Pooling is premature optimization — can be added later if profiling warrants it.

### DD-3: No VFX helper class
**Decision:** Inline `Object.Instantiate` / `Object.Destroy` directly in each ability's code.
**Rationale:** Only 3 lines per ability (null check + instantiate + schedule destroy). A helper wrapping Instantiate for 12 call sites is premature abstraction. If pooling is added later, that's when a helper gets introduced.

### DD-4: Sustained VFX — Instantiate on activate, Destroy on cleanup
**Decision:** Sustained abilities (MendingAura, IronGuard, Empower, ManaCharge) store the spawned GameObject reference. `TryActivate()` spawns, `Cleanup()` destroys.
**Rationale:** Maps directly to existing IPathAbility activate/cleanup lifecycle. No orphaned GameObjects. Toggle abilities destroy on deactivation and re-spawn on reactivation.

## VFX Specifications

### Character Color Palette
| Character | Primary Color | Hex |
|-----------|--------------|-----|
| Brutor | Orange | #FF8C00 |
| Slasher | Crimson Red | #DC143C |
| Mystica | Emerald Green | #50C878 |
| Viper | Gold Yellow | #FFD700 |

### Per-Ability VFX Details

| # | Path | AbilityId | VFX Type | Description | Duration | Max Particles |
|---|------|-----------|----------|-------------|----------|---------------|
| 1 | Warden | Warden_Provoke | Burst AoE | Orange radial pulse expanding from player (radius matches TAUNT_RANGE=5) | 0.5s | 30 |
| 2 | Bulwark | Bulwark_IronGuard | Sustained aura | Blue-orange shield glow on player, particles orbit body | While active | 20 |
| 3 | Guardian | Guardian_ShieldLink | Sustained beam | Gold beam/tether from player outward (placeholder — no ally target yet) | While active | 15 |
| 4 | Executioner | Executioner_MarkForDeath | Target marker | Red pulsing skull/crosshair indicator above marked target | 6s (mark duration) | 10 |
| 5 | Reaper | Reaper_CleavingStrikes | On-hit burst | Red arc/slash trail on cleave impact | 0.3s | 25 |
| 6 | Shadow | Shadow_PhaseDash | Trail effect | Purple afterimage particles along dash path | 0.6s | 40 |
| 7 | Sage | Sage_MendingAura | Sustained aura | Green heal particles rising from player feet | While active | 20 |
| 8 | Enchanter | Enchanter_Empower | Sustained aura | Cyan/teal buff glow wrapping player, sparkle particles | While active | 20 |
| 9 | Conjurer | Conjurer_SummonSproutling | Spawn burst | Green leafy burst at spawn position | 0.5s | 30 |
| 10 | Marksman | Marksman_PiercingShots | Passive indicator | Yellow arrow trail particles on player (passive active indicator) | While active | 15 |
| 11 | Trapper | Trapper_HarpoonShot | Projectile trail | Yellow chain-link trail from player to target | 0.4s | 25 |
| 12 | Arcanist | Arcanist_ManaCharge | Sustained charge | Purple energy spiral converging on player, intensifies with charge% | While channeling | 40 |

## File Plan

### New Files

#### 1. `Assets/Editor/AbilityVfxCreator.cs`
- Menu: "TomatoFighters → Create Ability VFX Prefabs"
- Creates `AbilityVfxLookup.asset` at `ScriptableObjects/VFX/`
- Creates 12 VFX prefabs at `Prefabs/Effects/Abilities/`
- Each prefab: empty GameObject → ParticleSystem component configured per spec above
- Sets color, emission rate, shape, max particles, duration, looping (for sustained) vs one-shot (for burst)
- Populates the lookup SO entries array with all 12 mappings

#### 2. `Assets/Scripts/Shared/Data/AbilityVfxLookup.cs`
- ScriptableObject with `Entry[]` (abilityId + vfxPrefab)
- `GetVfx(string abilityId)` lookup method
- Lives in Shared so both Combat and World pillars can reference it

### Modified Files

#### 3. `Assets/Scripts/Characters/Abilities/PathAbilityContext.cs`
Add one field:
```csharp
/// <summary>VFX prefab for this ability. Null = no VFX.</summary>
public GameObject VfxPrefab { get; set; }
```

#### 4. `Assets/Scripts/Characters/PathAbilityExecutor.cs`
- Add `[SerializeField] private AbilityVfxLookup vfxLookup;`
- When creating abilities, query `vfxLookup.GetVfx(abilityId)` and set `ctx.VfxPrefab`

#### 5–16. 12 T1 Ability Files
Each ability gets VFX spawn logic based on its type:

**Burst abilities** (Provoke, CleavingStrikes, SummonSproutling, HarpoonShot, PhaseDash):
```csharp
// In TryActivate():
if (_ctx.VfxPrefab != null)
    Object.Destroy(
        Object.Instantiate(_ctx.VfxPrefab, _ctx.PlayerTransform.position, Quaternion.identity),
        0.5f);
```

**Sustained abilities** (IronGuard, ShieldLink, MendingAura, Empower, PiercingShots, ManaCharge):
```csharp
private GameObject _activeVfx;

// In TryActivate():
if (_ctx.VfxPrefab != null)
    _activeVfx = Object.Instantiate(_ctx.VfxPrefab, _ctx.PlayerTransform.position, Quaternion.identity);

// In Cleanup():
if (_activeVfx != null)
    Object.Destroy(_activeVfx);
```

**Target-placed abilities** (MarkForDeath):
```csharp
// In TryActivate(), after finding the target:
if (_ctx.VfxPrefab != null && markedTarget != null)
    Object.Destroy(
        Object.Instantiate(_ctx.VfxPrefab, markedTarget.position, Quaternion.identity),
        MARK_DURATION);
```

## Execution Order

1. `AbilityVfxLookup.cs` — data class first (no dependencies)
2. `PathAbilityContext.cs` — add VfxPrefab field
3. `PathAbilityExecutor.cs` — wire up vfxLookup injection
4. 12 T1 ability files — add VFX spawn calls (can be done in any order)
5. `AbilityVfxCreator.cs` — Creator Script that generates all prefabs + SO asset

## Acceptance Criteria

- [ ] 12 VFX prefabs generated by Creator Script (one per T1 ability)
- [ ] Particle systems configured per spec (color, count, duration, shape)
- [ ] Color-coded per character (Brutor=orange, Slasher=red, Mystica=green, Viper=yellow)
- [ ] Performance: max 50 particles per effect
- [ ] AbilityVfxLookup SO maps all 12 ability IDs to prefabs
- [ ] All 12 T1 abilities spawn their VFX on activation
- [ ] Sustained abilities destroy VFX on deactivation/cleanup
- [ ] No cross-pillar import violations

## Risks & Gotchas

- **Guardian ShieldLink beam:** No ally target system exists yet. VFX will be a placeholder glow on player, not an actual beam to a teammate. Revisit when co-op is implemented.
- **MarkForDeath target position:** Needs access to the marked enemy transform. Currently the ability finds targets via Physics2D — VFX spawn at the hit collider position.
- **PhaseDash trail:** Trail effect spawns at player position on dash start. True motion trail (particles along path) would need TrailRenderer or per-frame emission — start with burst at dash origin.
- **ManaCharge intensity scaling:** The spec says "intensifies with charge%". Simplest approach: start at low emission rate, scale `emissionModule.rateOverTime` proportional to `ChargePercent` in Tick(). Requires caching the ParticleSystem reference.
- **Sustained VFX parenting:** Sustained effects should be parented to `PlayerTransform` so they follow the player. Use `Instantiate(prefab, playerTransform)` overload.
