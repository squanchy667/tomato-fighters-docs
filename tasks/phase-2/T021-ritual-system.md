# T021: RitualSystem — Trigger Pipeline

## Overview
| Field | Value |
|-------|-------|
| **Phase** | 2 |
| **Priority** | P1 |
| **Owner** | Dev 2 |
| **Depends on** | T020 (RitualData SO — DONE) |
| **Blocks** | T029 (RitualStackCalculator), T031 (RewardSelectorUI) |
| **Branch** | `pillar2/T021-ritual-system` |

## What We're Building
A `RitualSystem` MonoBehaviour that is the runtime backbone of the roguelite loop.
It subscribes to all `ICombatEvents` triggers, dispatches matching ritual effects via a
handler table, tracks per-family ritual counts, and implements `IBuffProvider` so Combat
can query damage/speed/defense multipliers on every hit.

## Acceptance Criteria
- [ ] Subscribes to all ICombatEvents triggers (13 events)
- [ ] Trigger → matching ritual → fire effect pipeline
- [ ] Add ritual (new or level-up existing)
- [ ] Family count tracking (per RitualFamily enum value)
- [ ] IBuffProvider implementation for damage/speed/defense multipliers

## File Plan

| File | Action | Notes |
|------|--------|-------|
| `Scripts/Shared/Data/PlaceholderTypes.cs` | **Update** | Replace `OnHitEffect` + `OnTriggerEffect` with single `RitualEffect` class (DD-1, DD-3) |
| `Scripts/Roguelite/RitualSystem.cs` | **Create** | Main MonoBehaviour — see spec below |

`ActiveRitualEntry` is a **nested private class** inside `RitualSystem.cs` (DD-6). No separate file.

## Design Decisions

### DD-1: `RitualEffect` shape — data-only class
`OnTriggerEffect` and `OnHitEffect` are replaced by a single `RitualEffect` class in
`Shared/Data/PlaceholderTypes.cs`. Fields Combat reads and applies — no delegates, no enum
discriminators.

```csharp
/// <summary>
/// Effect data returned by IBuffProvider. Combat applies this alongside or after an action.
/// Both on-hit and on-trigger effects share this type — the list they come from determines when
/// they fire (every hit vs. specific trigger).
/// </summary>
public class RitualEffect
{
    public DamageType damageType;     // elemental damage type to apply
    public float damageMultiplier;    // bonus damage as fraction of the triggering hit
    public GameObject vfxPrefab;      // spawned at hit position (null = no VFX)
    public float speedMultiplier;     // temporary speed change (1.0 = none)
}
```

`IBuffProvider` signatures update accordingly:
```csharp
List<RitualEffect> GetAdditionalOnHitEffects();
List<RitualEffect> GetTriggerEffects(RitualTrigger trigger);
```

### DD-2: Handler scope — infrastructure + VFX only
T021 implements the trigger pipeline and multiplier queries. Actual DoT damage logic
(Burn ticks, Chain Lightning bounces) is **deferred to Phase 3** — it requires stable
`IDamageable` target lifetime management which belongs to the World pillar.

Handlers in T021 do:
- Spawn `effectPrefab` at hit position (VFX)
- Return a `RitualEffect` with `damageMultiplier` for instant bonus elemental damage
- Update multiplier contributions for `IBuffProvider` queries

Handlers do NOT do:
- `StartCoroutine` for DoT ticks
- Call `IDamageable.TakeDamage` on a target

### DD-3: Single `RitualEffect` class for both lists
`GetAdditionalOnHitEffects()` and `GetTriggerEffects()` both return `List<RitualEffect>`.
The distinction between "fires every hit" vs. "fires on specific trigger" is captured by
which list the effect lives in — not the class type. No duplication, one class to maintain.

### DD-4: Multiplier caching — recalculate on every query
`GetDamageMultiplier`, `GetSpeedMultiplier`, `GetDefenseMultiplier` iterate `_activeRituals`
on each call. No dirty-flag cache. Rationale: ≤20 rituals max per run = ~20 float
multiplications per query. Cache correctness risk outweighs the immeasurable performance gain.

```csharp
public float GetDamageMultiplier(DamageType type)
{
    float mult = 1f;
    foreach (var entry in _activeRituals)
        mult *= entry.GetDamageContribution(type);
    return mult;
}
```

### DD-5: ICombatEvents injection — SerializeField MonoBehaviour
Same pattern as `PathSystem`. Null-safe cast in `Awake`/`OnDestroy`.

```csharp
[SerializeField] private MonoBehaviour _combatEventsSource;

private void Awake()
{
    var src = _combatEventsSource as ICombatEvents;
    if (src != null)
    {
        src.OnStrike     += HandleStrike;
        src.OnSkill      += HandleSkill;
        // ... all 13 events
    }
}
```

### DD-6: `ActiveRitualEntry` — nested private class
Only `RitualSystem` creates or reads `ActiveRitualEntry`. No other system references it
by type — they interact via public API (`AddRitual`, `LevelUpRitual`, `GetFamilyCount`).

```csharp
private class ActiveRitualEntry
{
    public RitualData data;
    public int level;           // 1-3
    public int currentStacks;   // for stack-based effects
    public float lastStackTime; // for stack decay (Phase 3)

    public float GetDamageContribution(DamageType type) { ... }
}
```

### DD-7: Instant ritual damage only — no DoT flag
`RitualEffect` has no `isDoT`, `dotDuration`, or `dotTickInterval` fields in T021.
All ritual damage is expressed as instant bonus elemental damage in the same frame.
Burn = instant Fire bonus hit. Chain Lightning = instant Lightning bonus hit.

This is a valid Phase 2 approximation — ritual power levels are tunable and testable.
DoT ticks are a Phase 3 feel/polish concern.

**Deferred hook in every damage handler:**
```csharp
// TODO [Phase 3 / T029+]: Replace with DoT coroutine when IDamageable target lifetime is stable.
// Add to RitualEffect: float dotDuration, float dotTickInterval.
// Handler: StartCoroutine(ElementalTick(ctx.target, effect)) instead of instant hit.
```

## Deferred (Intentional — Not Forgotten)
| What | When | Where tracked |
|------|------|---------------|
| DoT tick coroutines (Burn, etc.) | Phase 3, T029 | TODO comment in each damage handler + T029 description |
| Stack decay timers (`lastStackTime`) | Phase 3, T029 | `ActiveRitualEntry.lastStackTime` field exists, unused |
| Twin Ritual family synergy bonuses | T047 | `GetFamilyCount()` already exposed for T047 to read |

## `RitualSystem` Public API

```csharp
// Ritual management
public bool AddRitual(RitualData data);          // new ritual at level 1; returns false if already at max level
public bool LevelUpRitual(RitualData data);      // increments existing ritual level; returns false if not found
public void ResetForNewRun();                     // clears all active rituals

// Family tracking
public int GetFamilyCount(RitualFamily family);  // how many active rituals belong to this family

// IBuffProvider (implemented)
public float GetDamageMultiplier(DamageType type);
public float GetSpeedMultiplier();
public float GetDefenseMultiplier();
public List<RitualEffect> GetAdditionalOnHitEffects();
public List<RitualEffect> GetTriggerEffects(RitualTrigger trigger);
public bool IsRepetitivePenaltyOverridden();
public float GetPathDamageMultiplier();    // always 1.0 — path bonuses are PathSystem's job
public float GetPathDefenseMultiplier();   // always 1.0
public float GetPathSpeedMultiplier();     // always 1.0
public List<PathAbility> GetActivePathAbilities(); // always empty — PathSystem's job
```

## Handler Dispatch Table

Handler key = `RitualData.effectId` string. Registered in `Awake`.

Initial handlers to implement (Fire + Lightning families from T020):

| effectId | Trigger | Instant Effect |
|----------|---------|----------------|
| `Fire_Burn` | OnStrike | +Fire damageMultiplier, fire VFX |
| `Fire_BlazingDash` | OnDash | +Fire damageMultiplier, blaze VFX |
| `Fire_FlameStrike` | OnSkill | +Fire damageMultiplier, flame VFX |
| `Fire_EmberShield` | OnTakeDamage | +defense speedMultiplier, ember VFX |
| `Lightning_ChainLightning` | OnStrike | +Lightning damageMultiplier, chain VFX |
| `Lightning_LightningStrike` | OnFinisher | +Lightning damageMultiplier, strike VFX |
| `Lightning_ShockWave` | OnKill | +Lightning damageMultiplier, wave VFX |
| `Lightning_StaticField` | OnDash | +speedMultiplier, static VFX |

Unknown effectId → `Debug.LogWarning` and skip. Never throw.

## Execution Order

1. Update `PlaceholderTypes.cs` — define `RitualEffect`, update `IBuffProvider` signatures
2. Write `ActiveRitualEntry` nested class inside `RitualSystem.cs`
3. Write `RitualSystem` skeleton — MonoBehaviour + `IBuffProvider` shell returning defaults
4. Wire all 13 `ICombatEvents` subscriptions
5. Implement handler dispatch table with Fire + Lightning handlers
6. Implement `GetDamageMultiplier`, `GetSpeedMultiplier`, `GetDefenseMultiplier`
7. Implement `GetAdditionalOnHitEffects` + `GetTriggerEffects`
8. Implement `AddRitual`, `LevelUpRitual`, `ResetForNewRun`, `GetFamilyCount`
9. Verify `IBuffProvider` is fully satisfied (no unimplemented members)
