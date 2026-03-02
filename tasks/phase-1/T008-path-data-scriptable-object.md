# T008: PathData ScriptableObject — 12 Paths

**Phase:** 1 | **Priority:** P0 | **Owner:** Dev 2 | **Status:** PENDING
**Depends on:** T001 ✓, T006 ✓
**Blocks:** T018 (PathSystem), T028 (Path T1 Ability Execution — Dev 1)

---

## Summary

Define the `PathData` ScriptableObject and create all 12 path assets with the correct stat bonuses and ability IDs from CHARACTER-ARCHETYPES.md. This is the data foundation for the entire path upgrade system.

---

## Deliverables

| File | Description |
|------|-------------|
| `Assets/Scripts/Shared/Data/PathData.cs` | SO definition + `PathTierBonuses` struct |
| `Assets/Scripts/Shared/Data/PathTierBonuses.cs` | Serializable struct for per-tier stat deltas |
| `Assets/Editor/PathDataCreator.cs` | MenuItem editor script — creates all 12 assets |
| `Assets/Scripts/Shared/Interfaces/IPathProvider.cs` | Update `object MainPath/SecondaryPath` → `PathData` |
| `ScriptableObjects/Paths/Brutor/WardenPath.asset` | |
| `ScriptableObjects/Paths/Brutor/BulwarkPath.asset` | |
| `ScriptableObjects/Paths/Brutor/GuardianPath.asset` | |
| `ScriptableObjects/Paths/Slasher/ExecutionerPath.asset` | |
| `ScriptableObjects/Paths/Slasher/ReaperPath.asset` | |
| `ScriptableObjects/Paths/Slasher/ShadowPath.asset` | |
| `ScriptableObjects/Paths/Mystica/SagePath.asset` | |
| `ScriptableObjects/Paths/Mystica/EnchanterPath.asset` | |
| `ScriptableObjects/Paths/Mystica/ConjurerPath.asset` | |
| `ScriptableObjects/Paths/Viper/MarksmanPath.asset` | |
| `ScriptableObjects/Paths/Viper/TrapperPath.asset` | |
| `ScriptableObjects/Paths/Viper/ArcanistPath.asset` | |

---

## Acceptance Criteria

- [ ] `PathData` SO with `PathType`, `CharacterType`, description
- [ ] `PathTierBonuses` serializable struct with named fields for all stats
- [ ] Tier 3 visually marked as Main Path Only via `[Header]` in Inspector
- [ ] `GetStatBonusArray(int tier)` returns cumulative `float[]` indexed by `(int)StatType`
- [ ] `IPathProvider.MainPath` and `SecondaryPath` updated from `object` to `PathData`
- [ ] All 12 path assets created via Editor MenuItem with values from CHARACTER-ARCHETYPES.md
- [ ] Compiles with zero warnings

---

## Design Decisions

### DD-1: PathTierBonuses as Named Struct

**Decision:** Tier bonuses are stored as a `[Serializable]` struct with named fields per stat, not as a raw `float[]` array.

**Rationale:** Named fields prevent silent index errors in the Inspector, are readable to designers without needing to count array indices, and match the stat names used in CHARACTER-ARCHETYPES.md exactly.

```csharp
[Serializable]
public struct PathTierBonuses
{
    public int   healthBonus;
    public int   defenseBonus;
    public float attackBonus;
    public float rangedAttackBonus;   // Viper-path bonuses only; 0 on all Brutor/Slasher/Mystica paths
    public float throwableAttackBonus;
    public float speedBonus;
    public int   manaBonus;
    public float manaRegenBonus;
    public float critChanceBonus;     // Stored as decimal: 0.05 = 5%
    public float stunRateBonus;       // "PRS" in design docs
}
```

### DD-2: T3 Marked via [Header], No Runtime Bool

**Decision:** Tier 3 fields are separated by `[Header("Tier 3 — Main Path Only")]` in the Inspector. No `bool isMainExclusive` field exists on the SO.

**Rationale:** Every path's T3 is Main-only by design — a bool would always be `true` across all 12 assets, adding noise. PathSystem (T018) enforces the constraint in behavior. The header self-documents the constraint to any designer opening the asset without runtime overhead.

```csharp
[Header("Tier 3 — Main Path Only")]
public PathTierBonuses tier3Bonuses;
public string tier3AbilityId;
```

### DD-3: Incremental Tier Bonuses

**Decision:** Each tier struct stores the *delta that tier adds*, not a cumulative total. `GetStatBonusArray(int tier)` sums the deltas up to the requested tier.

**Rationale:** The design doc lists per-tier values as increments ("T2: +30 HP, +0.3 PRS"). Storing increments keeps assets honest and transparent. The summation logic lives once in `GetStatBonusArray` rather than being implicit knowledge about how to read cumulative values.

```csharp
public float[] GetStatBonusArray(int tier)
{
    var bonuses = new float[StatModifierInput.StatCount];
    if (tier >= 1) Accumulate(tier1Bonuses, bonuses);
    if (tier >= 2) Accumulate(tier2Bonuses, bonuses);
    if (tier >= 3) Accumulate(tier3Bonuses, bonuses);
    return bonuses;
}
```

---

## Execution Order

1. **`PathTierBonuses.cs`** — Serializable struct (no Unity deps, compiles immediately)
2. **`PathData.cs`** — SO definition referencing the struct + `GetStatBonusArray`
3. **`IPathProvider.cs`** — Swap `object` placeholders for `PathData` (unblocks Dev 1 + Dev 3)
4. **`PathDataCreator.cs`** — Editor MenuItem; populates all 12 assets with values from CHARACTER-ARCHETYPES.md
5. **Run the MenuItem in Unity Editor** → verify 12 assets appear at `ScriptableObjects/Paths/`

---

## Path Stat Bonus Reference

All values are **incremental deltas per tier** (each tier adds to the previous).

### Brutor

| Path | Tier | HP | DEF | ATK | SPD | MNA | MRG | CRT | PRS |
|------|------|----|-----|-----|-----|-----|-----|-----|-----|
| Warden | T1 | +20 | — | — | — | — | — | — | +0.2 |
| | T2 | +30 | — | +0.1 | — | — | — | — | +0.3 |
| | T3 | +40 | — | +0.2 | — | — | — | — | +0.5 |
| Bulwark | T1 | +30 | +5 | — | — | — | — | — | — |
| | T2 | +40 | +8 | — | — | — | — | — | — |
| | T3 | +50 | +12 | — | — | — | — | +5% | — |
| Guardian | T1 | +25 | +3 | — | — | +20 | — | — | — |
| | T2 | +35 | +5 | — | — | +30 | — | — | — |
| | T3 | +45 | +8 | — | — | +40 | +2 | — | — |

### Slasher

| Path | Tier | HP | ATK | SPD | CRT | PRS |
|------|------|----|-----|-----|-----|-----|
| Executioner | T1 | — | +0.3 | — | +5% | — |
| | T2 | — | +0.4 | — | +8% | — |
| | T3 | — | +0.5 | — | +12% | +0.3 |
| Reaper | T1 | +15 | +0.2 | — | — | — |
| | T2 | +20 | +0.3 | +0.1 | — | — |
| | T3 | +30 | +0.4 | +0.2 | — | — |
| Shadow | T1 | — | — | +0.2 | +5% | — |
| | T2 | +10 | — | +0.3 | +8% | — |
| | T3 | +20 | — | +0.4 | +12% | — |

### Mystica

| Path | Tier | HP | ATK | SPD | MNA | MRG |
|------|------|----|-----|-----|-----|-----|
| Sage | T1 | +30 | — | — | +20 | +2 |
| | T2 | +40 | — | — | +30 | +3 |
| | T3 | +50 | — | — | +40 | +4 |
| Enchanter | T1 | — | — | +0.1 | +20 | +2 |
| | T2 | +15 | — | +0.1 | +30 | +3 |
| | T3 | +25 | — | +0.2 | +40 | +4 |
| Conjurer | T1 | +20 | +0.1 | — | +20 | — |
| | T2 | +30 | +0.2 | — | +30 | — |
| | T3 | +40 | +0.3 | — | +40 | +2 |

### Viper

| Path | Tier | HP | RATK | SPD | MNA | MRG | CRT | PRS |
|------|------|----|------|-----|-----|-----|-----|-----|
| Marksman | T1 | — | +0.3 | — | — | — | +5% | — |
| | T2 | +10 | +0.4 | — | — | — | +8% | — |
| | T3 | — | +0.5 | +0.1 | — | — | +12% | — |
| Trapper | T1 | +20 | — | +0.1 | — | — | — | +0.2 |
| | T2 | +30 | — | +0.2 | — | — | — | +0.3 |
| | T3 | +40 | — | +0.2 | +15 | — | — | +0.5 |
| Arcanist | T1 | — | — | — | +30 | +3 | — | — |
| | T2 | — | +0.2 | — | +40 | +4 | — | — |
| | T3 | — | +0.3 | — | +50 | +5 | +5% | — |

---

## Ability ID Reference

| Path | T1 ID | T2 ID | T3 ID |
|------|-------|-------|-------|
| Warden | `Warden_Provoke` | `Warden_AggroAura` | `Warden_WrathOfTheWarden` |
| Bulwark | `Bulwark_IronGuard` | `Bulwark_Retaliation` | `Bulwark_Fortress` |
| Guardian | `Guardian_ShieldLink` | `Guardian_RallyingPresence` | `Guardian_AegisDome` |
| Executioner | `Executioner_MarkForDeath` | `Executioner_ExecutionThreshold` | `Executioner_Deathblow` |
| Reaper | `Reaper_CleavingStrikes` | `Reaper_ChainSlash` | `Reaper_Whirlwind` |
| Shadow | `Shadow_PhaseDash` | `Shadow_Afterimage` | `Shadow_ThousandCuts` |
| Sage | `Sage_MendingAura` | `Sage_PurifyingBurst` | `Sage_Resurrection` |
| Enchanter | `Enchanter_Empower` | `Enchanter_ElementalInfusion` | `Enchanter_ArcaneOverdrive` |
| Conjurer | `Conjurer_SummonSproutling` | `Conjurer_DeployTotem` | `Conjurer_SummonGolem` |
| Marksman | `Marksman_PiercingShots` | `Marksman_RapidFire` | `Marksman_Killshot` |
| Trapper | `Trapper_HarpoonShot` | `Trapper_TrapNet` | `Trapper_AnchorChain` |
| Arcanist | `Arcanist_ManaCharge` | `Arcanist_ManaBlast` | `Arcanist_ManaOverload` |
