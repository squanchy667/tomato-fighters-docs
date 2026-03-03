# T020: RitualData ScriptableObject + Families

## Metadata

| Field | Value |
|-------|-------|
| **ID** | T020 |
| **Phase** | 2 |
| **Type** | Implementation |
| **Priority** | P1 |
| **Owner** | Dev 2 |
| **Branch** | `pillar2/T020-ritual-data-so` |
| **Depends On** | T001 |
| **Blocks** | T021 (RitualSystem) |

## Summary

Define the `RitualData` ScriptableObject — the authoritative data container for all ritual effects in the game. Rituals form the core of the roguelite modifier system: each run, the player collects rituals from 8 elemental families that trigger on combat actions and modify stats or apply effects.

This task creates the data layer only. The execution pipeline (`RitualSystem`) is T021.

## Requirements

1. `RitualData` ScriptableObject with all identity and scaling fields
2. `RitualLevelData` serializable struct for per-level power values
3. `RitualDataCreator` editor MenuItem to generate initial ritual assets
4. Fire family assets: Burn, Blazing Dash, Flame Strike, Ember Shield
5. Lightning family assets: Chain Lightning, Lightning Strike, Shock Wave, Static Field

## File Plan

| File | Action | Notes |
|------|--------|-------|
| `Assets/Scripts/Roguelite/RitualData.cs` | Create | SO + RitualLevelData struct |
| `Assets/Editor/RitualDataCreator.cs` | Create | MenuItem tool — Fire + Lightning families |

> **Note on placement:** Task board listed `Shared/Data/RitualData.cs`. Overridden by DD-3: no Shared interface exposes `RitualData` directly (IBuffProvider returns `OnTriggerEffect`, not `RitualData`), so Roguelite/ is the correct owner.

## Design Decisions

### DD-1: Effect type as `string effectId`
**Decision:** Use `string effectId` (e.g. `"Fire_Burn"`, `"Lightning_Chain"`) rather than a `RitualEffectType` enum or implicit family+trigger inference.

**Rationale:**
- Matches the established pattern from `PathData` (`tier1AbilityId`, `tier2AbilityId`, etc.)
- 36+ rituals across 8 families would bloat an enum with constant churn
- RitualSystem (T021) dispatches via `Dictionary<string, Action<RitualEffectContext>>`
- Designer adds a ritual SO and engineer wires the handler — clean separation

```csharp
// RitualSystem (T021) dispatch pattern:
private readonly Dictionary<string, Action<RitualEffectContext>> _handlers = new()
{
    ["Fire_Burn"]        = ctx => ApplyBurn(ctx),
    ["Lightning_Chain"]  = ctx => ApplyChainLightning(ctx),
};
```

### DD-2: Three explicit `RitualLevelData` structs
**Decision:** Three named level fields (`level1`, `level2`, `level3`) rather than a single base value + fixed constants or a `float[]` array.

**Rationale:**
- Mirrors `PathTierBonuses` pattern from T008 — consistent authoring model
- Inspector shows all three levels side-by-side — designer-friendly
- Level multipliers (1.0 / 1.5 / 2.0) are fixed in `RitualStackCalculator` (T029); per-level structs allow `baseValue` + `ritualPower` overrides where needed
- Avoids silent index errors from raw arrays

```csharp
[Serializable]
public struct RitualLevelData
{
    public float baseValue;       // raw effect magnitude
    public int   maxStacks;       // max simultaneous stacks
    public float stackingMultiplier; // bonus per stack (multiplicative)
    public float ritualPower;     // designer tuning knob
}
```

### DD-3: Lives in `Roguelite/`, not `Shared/Data/`
**Decision:** `RitualData` is in `Assets/Scripts/Roguelite/`, compiled under `TomatoFighters.Roguelite`.

**Rationale:**
- `IPathProvider.MainPath` returns `PathData` → PathData must be Shared.
- `IBuffProvider.GetTriggerEffects(RitualTrigger)` returns `List<OnTriggerEffect>` → RitualData never crosses the pillar boundary.
- Roguelite owns all ritual logic; keeping it in Roguelite/ enforces this ownership.

## RitualData Fields Reference

```
Identity
  ritualName       string
  description      string (TextArea)
  family           RitualFamily
  category         RitualCategory
  trigger          RitualTrigger

Effect
  effectId         string           (DD-1)
  effectPrefab     GameObject       (VFX — optional, assigned in Inspector)

Twin Ritual
  isTwin           bool
  secondFamily     RitualFamily     (only relevant when isTwin)

Level Scaling                       (DD-2)
  level1           RitualLevelData
  level2           RitualLevelData
  level3           RitualLevelData
```

## Initial Ritual Assets

### Fire Family

| Ritual | Category | Trigger | effectId | baseValue (L1/2/3) | maxStacks | stackMult | ritualPower |
|--------|----------|---------|----------|--------------------|-----------|-----------|-------------|
| Burn | Core | OnStrike | Fire_Burn | 5 / 5 / 5 | 3 / 4 / 5 | 1.2 / 1.25 / 1.3 | 1.0 |
| Blazing Dash | General | OnDash | Fire_BlazingDash | 15 / 15 / 15 | 1 | 1.0 | 1.0 / 1.5 / 2.0 |
| Flame Strike | Enhancement | OnFinisher | Fire_FlameStrike | 30 / 30 / 30 | 1 | 1.0 | 1.0 / 1.5 / 2.0 |
| Ember Shield | Enhancement | OnDeflect | Fire_EmberShield | 10 / 10 / 10 | 2 / 3 / 4 | 1.15 / 1.2 / 1.25 | 1.0 |

### Lightning Family

| Ritual | Category | Trigger | effectId | baseValue (L1/2/3) | maxStacks | stackMult | ritualPower |
|--------|----------|---------|----------|--------------------|-----------|-----------|-------------|
| Chain Lightning | Core | OnStrike | Lightning_Chain | 8 / 8 / 8 | 3 / 4 / 5 | 1.1 / 1.15 / 1.2 | 1.0 |
| Lightning Strike | General | OnSkill | Lightning_Strike | 25 / 25 / 25 | 1 | 1.0 | 1.0 / 1.5 / 2.0 |
| Shock Wave | Enhancement | OnFinisher | Lightning_ShockWave | 20 / 20 / 20 | 1 | 1.0 | 1.0 / 1.5 / 2.0 |
| Static Field | Enhancement | OnTakeDamage | Lightning_StaticField | 5 / 5 / 5 | 4 / 5 / 6 | 1.1 / 1.15 / 1.2 | 1.0 |

## Acceptance Criteria

- [ ] `RitualData` SO with all fields (identity, effect, twin, level scaling)
- [ ] `RitualLevelData` struct with baseValue, maxStacks, stackingMultiplier, ritualPower
- [ ] `CreateAssetMenu` path: `"TomatoFighters/Data/RitualData"`
- [ ] `RitualDataCreator` MenuItem: `"TomatoFighters/Create Ritual Assets"`
- [ ] Fire family: 4 assets created by MenuItem
- [ ] Lightning family: 4 assets created by MenuItem
- [ ] All assets at `Assets/ScriptableObjects/Rituals/{Family}/{Name}Ritual.asset`
- [ ] Compiles with zero warnings
- [ ] Namespace: `TomatoFighters.Roguelite`
