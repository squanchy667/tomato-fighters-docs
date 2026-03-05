# T017 — Character Passives

**Phase:** 2 (Core Combat + Path Framework)
**Pillar:** Combat (Dev 1)
**Priority:** P1
**Status:** DONE
**Branch:** `combat/T017-character-passives`
**Completed:** 2026-03-04
**Depends on:** T007 (CharacterStatCalculator), T014 (ComboSystem — All 4 Characters)

## Summary

Implement 4 character passive abilities that modify combat stats in real-time. Each passive reinforces the character's gameplay identity — Brutor's tankiness, Slasher's aggression, Mystica's support casting, Viper's ranged positioning.

## Passive Specifications

### 1. Brutor — "Thick Skin"
- **Type:** Always-on flat reduction
- 15% damage reduction from all sources (defense multiplier = 0.85)
- 40% knockback reduction (knockback multiplier = 0.6)
- No state management — constant values

### 2. Slasher — "Bloodlust"
- **Type:** Stacking ATK multiplier
- +3% ATK per hit landed (stack counter 0–10)
- 10 stacks max = +30% ATK
- Stacks reset to 0 if no hits for 3 seconds (timer only — combo drops do NOT reset)
- Multiplier: `1.0 + (0.03 × stackCount)`

### 3. Mystica — "Arcane Resonance"
- **Type:** Self-buff on ability cast (self-only until co-op ships)
- +5% damage per cast, stacks up to 3 times
- Each stack has independent 3-second expiry timer
- Multiplicative stacking: `1.05^activeStacks` (3 stacks ≈ 1.157x)
- Triggers on: Light/Heavy attacks, path abilities (any attack event)

### 4. Viper — "Distance Bonus"
- **Type:** Range-based per-hit scaling
- +2% damage per unit distance to target at hit time
- Max +30% (capped at 15 units)
- Multiplier: `1.0 + min(0.02 × distance, 0.3)`
- Distance passed via HitContext struct — passive does not access transforms

## Design Decisions

### DD-1: PassiveConfig ScriptableObject + Plain C# Logic
**Rationale:** Follows both "ScriptableObjects for ALL data" and "plain C# for testable logic" conventions. Tunable values (stack cap, decay time, DR %) live in a PassiveConfig SO editable from Inspector. Logic classes receive config and are unit-testable without Unity.

```csharp
[CreateAssetMenu(menuName = "TomatoFighters/PassiveConfig")]
public class PassiveConfig : ScriptableObject
{
    [Header("Thick Skin")]
    public float thickSkinDamageReduction = 0.15f;
    public float thickSkinKnockbackReduction = 0.40f;

    [Header("Bloodlust")]
    public float bloodlustAtkPerStack = 0.03f;
    public int bloodlustMaxStacks = 10;
    public float bloodlustDecayTime = 3f;

    [Header("Arcane Resonance")]
    public float arcaneResonanceDmgPerStack = 0.05f;
    public int arcaneResonanceMaxStacks = 3;
    public float arcaneResonanceStackDuration = 3f;

    [Header("Distance Bonus")]
    public float distanceBonusPerUnit = 0.02f;
    public float distanceBonusMaxPercent = 0.30f;
}
```

### DD-2: Self-Buff Only for Arcane Resonance
**Rationale:** Co-op doesn't exist yet. Implementing ally-broadcast now would be speculative. When co-op ships, the system will likely need redesigning anyway. Mystica buffs herself on cast — add ally broadcast later.

### DD-3: HitContext Struct for Distance and Per-Hit Data
**Rationale:** Keeps passives decoupled from transforms and scene queries. The hit handler calculates distance and passes context. Future-proofs for any passive that needs hit circumstances.

```csharp
public struct HitContext
{
    public DamageType damageType;
    public float distanceToTarget;
    public bool isPunishHit;
}
```

### DD-4: IPassiveProvider Interface (Separate from IBuffProvider)
**Rationale:** IBuffProvider is Roguelite's channel to Combat. Passives are a Combat concern. Dedicated interface avoids overloading IBuffProvider's purpose and lets both systems evolve independently.

```csharp
public interface IPassiveProvider
{
    float GetDamageMultiplier(HitContext context);
    float GetDefenseMultiplier();
    float GetKnockbackMultiplier();
    float GetSpeedMultiplier();
    void Tick(float deltaTime);
}
```

Damage formula becomes:
```
finalDmg = baseATK × attackMult × IBuffProvider.GetDamageMultiplier()
                                × IPassiveProvider.GetDamageMultiplier(hitContext)
```

### DD-5: Bloodlust Decay — Timer Only
**Rationale:** 3-second timer resets stacks to 0. Combo drops do NOT reset stacks. Double-punishing (lost combo chain + lost stacks) is too harsh. Slasher's fast 3-hit chain means maintaining hits within 3s is the real skill test.

## File Plan

| # | File | Type | Purpose |
|---|------|------|---------|
| 1 | `Scripts/Shared/Data/HitContext.cs` | Struct | Per-hit context (damage type, distance, punish) |
| 2 | `Scripts/Shared/Interfaces/IPassiveProvider.cs` | Interface | Passive multiplier queries for damage pipeline |
| 3 | `Scripts/Characters/Passives/PassiveConfig.cs` | ScriptableObject | Tunable values for all 4 passives |
| 4 | `Scripts/Characters/Passives/IPassiveAbility.cs` | Interface | Contract for individual passive implementations |
| 5 | `Scripts/Characters/Passives/ThickSkin.cs` | Plain C# | Brutor passive — flat DR + knockback reduction |
| 6 | `Scripts/Characters/Passives/Bloodlust.cs` | Plain C# | Slasher passive — stacking ATK on hits |
| 7 | `Scripts/Characters/Passives/ArcaneResonance.cs` | Plain C# | Mystica passive — self-buff on cast |
| 8 | `Scripts/Characters/Passives/DistanceBonus.cs` | Plain C# | Viper passive — range-based damage scaling |
| 9 | `Scripts/Characters/PassiveAbilitySystem.cs` | MonoBehaviour | Holds active passive, subscribes to events, implements IPassiveProvider |
| 10 | `Editor/Characters/*CharacterCreator.cs` | Editor (modify) | Wire PassiveAbilitySystem + PassiveConfig onto player prefabs |
| 11 | `ScriptableObjects/Passives/PassiveConfig.asset` | Asset (created by creator) | Default tuning values |
| 12 | `Tests/EditMode/Characters/BloodlustTests.cs` | Test | Stack increment, decay, cap |
| 13 | `Tests/EditMode/Characters/DistanceBonusTests.cs` | Test | Distance scaling, cap |
| 14 | `Tests/EditMode/Characters/ThickSkinTests.cs` | Test | Flat multiplier values |
| 15 | `Tests/EditMode/Characters/ArcaneResonanceTests.cs` | Test | Stack/expiry/multiplicative stacking |

## Execution Order

1. `HitContext` struct + `IPassiveProvider` interface (Shared)
2. `PassiveConfig` ScriptableObject
3. `IPassiveAbility` interface
4. 4 passive classes: ThickSkin → Bloodlust → DistanceBonus → ArcaneResonance
5. `PassiveAbilitySystem` MonoBehaviour
6. Wire into Character Creator scripts
7. Edit mode tests

## Acceptance Criteria

- [ ] 4 passive abilities implemented as plain C# with PassiveConfig SO
- [ ] IPassiveProvider interface in Shared/Interfaces
- [ ] HitContext struct in Shared/Data
- [ ] PassiveAbilitySystem MonoBehaviour on player prefab
- [ ] Bloodlust: stacks increment on hit, decay after 3s, cap at 10
- [ ] ThickSkin: constant 0.85 damage mult, 0.6 knockback mult
- [ ] ArcaneResonance: self-buff stacks on cast, independent 3s expiry, multiplicative
- [ ] DistanceBonus: per-hit distance scaling, capped at 1.3x
- [ ] Edit mode tests for all 4 passives
- [ ] Creator scripts updated to wire PassiveAbilitySystem
