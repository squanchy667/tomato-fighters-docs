# T001: Shared Interfaces, Enums, and Data Structures

## Metadata
| Field | Value |
|-------|-------|
| **Phase** | 1 — Foundation |
| **Type** | implementation |
| **Priority** | P0 |
| **Owner** | ALL |
| **Agent** | shared-contracts |
| **Depends On** | none |
| **Blocks** | T002, T003, T005, T006, T008, T009, T010, T011, T012 |
| **Status** | PENDING |
| **Branch** | `shared/T001-shared-contracts` |

## Objective
Define every cross-pillar interface contract, shared enum, and shared data structure that the 3-pillar architecture depends on. This is the foundational layer that enables all three developers to work independently on Combat, Roguelite, and World pillars without importing from each other's namespaces.

## Context
Tomato Fighters uses a strict 3-pillar architecture (Combat / Roguelite / World) where **all cross-domain communication flows through `Shared/Interfaces/`**. No pillar may import from another pillar's namespace — ever. This task produces the contracts that every subsequent task depends on. It is a sync session: all 3 devs must agree on every interface signature before implementation proceeds.

Reference the full interface specs in `architecture/interface-contracts.md` and the data flow diagrams in `architecture/data-flow.md`. Character stats and path enums come from `design-specs/CHARACTER-ARCHETYPES.md`.

## Requirements

### Interfaces (6 files)

1. **ICombatEvents** (`Shared/Interfaces/ICombatEvents.cs`) — Combat fires, Roguelite subscribes
   - 13 event signatures using `Action<T>`: `OnStrike`, `OnSkill`, `OnArcana`, `OnDash`, `OnDeflect`, `OnClash`, `OnPunish`, `OnKill`, `OnFinisher`, `OnJump`, `OnDodge`, `OnTakeDamage`, `OnPathAbilityUsed`
   - Each event uses its own data struct (`StrikeEventData`, `SkillEventData`, etc.)

2. **IBuffProvider** (`Shared/Interfaces/IBuffProvider.cs`) — Roguelite provides, Combat queries
   - `float GetDamageMultiplier(DamageType type)`
   - `float GetSpeedMultiplier()`
   - `float GetDefenseMultiplier()`
   - `List<OnHitEffect> GetAdditionalOnHitEffects()`
   - `List<OnTriggerEffect> GetTriggerEffects(RitualTrigger trigger)`
   - `bool IsRepetitivePenaltyOverridden()`
   - `float GetPathDamageMultiplier()`
   - `float GetPathDefenseMultiplier()`
   - `float GetPathSpeedMultiplier()`
   - `List<PathAbility> GetActivePathAbilities()`

3. **IPathProvider** (`Shared/Interfaces/IPathProvider.cs`) — Roguelite provides, Combat + World query
   - `CharacterType Character { get; }`
   - `PathData MainPath { get; }`
   - `PathData SecondaryPath { get; }`
   - `int MainPathTier { get; }` (1-3)
   - `int SecondaryPathTier { get; }` (1-2)
   - `bool HasPath(PathType type)`
   - `float GetPathStatBonus(StatType stat)`
   - `bool IsPathAbilityUnlocked(string abilityId)`

4. **IDamageable** (`Shared/Interfaces/IDamageable.cs`) — Combat defines, World implements on enemies
   - `void TakeDamage(DamagePacket damage)`
   - `float CurrentHealth { get; }`
   - `float MaxHealth { get; }`
   - `void AddPressure(float amount)`
   - `bool IsStunned { get; }`
   - `void ApplyKnockback(Vector2 force)`
   - `void ApplyLaunch(Vector2 force)`
   - `bool IsInvulnerable { get; }`

5. **IAttacker** (`Shared/Interfaces/IAttacker.cs`) — World implements on enemies, Combat reads
   - `AttackData CurrentAttack { get; }`
   - `bool IsCurrentAttackUnstoppable { get; }`
   - `TelegraphType CurrentTelegraphType { get; }`
   - `float PunishWindowDuration { get; }`
   - `bool IsInPunishableState { get; }`

6. **IRunProgressionEvents** (`Shared/Interfaces/IRunProgressionEvents.cs`) — World fires, Roguelite subscribes
   - `event Action<AreaClearedData> OnAreaCleared`
   - `event Action<BossDefeatedData> OnBossDefeated`
   - `event Action<IslandCompletedData> OnIslandCompleted`
   - `event Action<ShopEnteredData> OnShopEntered`
   - `event Action OnRunStarted`
   - `event Action<RunEndData> OnRunEnded`
   - `event Action<PathSelectedData> OnMainPathSelected`
   - `event Action<PathSelectedData> OnSecondaryPathSelected`
   - `event Action<PathTierUpData> OnPathTierUp`

### Enums (1 file or split by domain)

7. **CharacterType** — `Brutor, Slasher, Mystica, Viper`
8. **PathType** — `Warden, Bulwark, Guardian, Executioner, Reaper, Shadow, Sage, Enchanter, Conjurer, Marksman, Trapper, Arcanist`
9. **StatType** — `Health, Defense, Attack, RangedAttack, Speed, Mana, ManaRegen, CritChance, PressureRate`
10. **DamageType** — `Physical, Fire, Lightning, Water, Thorn, Gale, Time, Cosmic, Necro`
11. **RitualTrigger** — `OnStrike, OnSkill, OnDash, OnDeflect, OnClash, OnFinisher, OnKill, OnArcana, OnJump, OnDodge, OnTakeDamage`
12. **TelegraphType** — `Normal, Unstoppable`
13. **RitualFamily** — `Fire, Lightning, Water, Thorn, Gale, Time, Cosmic, Necro`
14. **RitualCategory** — `Core, General, Enhancement, Twin`

### Data Structures

15. **DamagePacket** (`Shared/Data/DamagePacket.cs`) — struct with: `DamageType type`, `float amount`, `bool isPunishDamage`, `Vector2 knockbackForce`, `Vector2 launchForce`, `CharacterType source`
16. **Event data structs** — `StrikeEventData`, `SkillEventData`, `ArcanaEventData`, `DashEventData`, `DeflectEventData`, `ClashEventData`, `PunishEventData`, `KillEventData`, `FinisherEventData`, `JumpEventData`, `DodgeEventData`, `TakeDamageEventData`, `PathAbilityEventData` (each with contextually relevant fields)
17. **Run event data structs** — `AreaClearedData`, `BossDefeatedData`, `IslandCompletedData`, `ShopEnteredData`, `RunEndData`, `PathSelectedData`, `PathTierUpData`
18. **Placeholder types** — `OnHitEffect`, `OnTriggerEffect`, `PathAbility` (can be empty classes/structs that get fleshed out in later tasks)

## File Plan

| File Path | Description |
|-----------|-------------|
| `Shared/Interfaces/ICombatEvents.cs` | 13-event interface for combat-to-roguelite communication |
| `Shared/Interfaces/IBuffProvider.cs` | 10-method interface for roguelite-to-combat stat queries |
| `Shared/Interfaces/IPathProvider.cs` | 8-member interface for path system queries |
| `Shared/Interfaces/IDamageable.cs` | 8-member interface for damage/health/knockback on entities |
| `Shared/Interfaces/IAttacker.cs` | 5-member interface for attack state queries |
| `Shared/Interfaces/IRunProgressionEvents.cs` | 9-event interface for run progression communication |
| `Shared/Enums/CharacterType.cs` | Character enum (4 values) |
| `Shared/Enums/PathType.cs` | Path enum (12 values, grouped by character) |
| `Shared/Enums/StatType.cs` | Stat enum (9 values) |
| `Shared/Enums/DamageType.cs` | Damage type enum (9 values) |
| `Shared/Enums/RitualTrigger.cs` | Ritual trigger enum (11 values) |
| `Shared/Enums/TelegraphType.cs` | Telegraph enum (2 values) |
| `Shared/Enums/RitualFamily.cs` | Ritual family enum (8 values) |
| `Shared/Enums/RitualCategory.cs` | Ritual category enum (4 values) |
| `Shared/Data/DamagePacket.cs` | Damage payload struct |
| `Shared/Data/AttackData.cs` | Empty SO stub (filled in T005) |
| `Shared/Data/PathData.cs` | Empty SO stub (filled in T008) |
| `Shared/Data/CombatEventData.cs` | All 13 combat event data structs |
| `Shared/Data/RunEventData.cs` | All 7 run progression event data structs |
| `Shared/Data/PlaceholderTypes.cs` | OnHitEffect, OnTriggerEffect, PathAbility stubs |

## Design Decisions

These decisions were agreed during task planning:

### DD-1: Event data structs → `readonly struct`
Combat events (`OnStrike`, etc.) fire every attack frame. Using `readonly struct` avoids GC allocations in the hot path. Subscribers that need to modify data do so in their own scope — event data is fire-and-forget.

```csharp
public readonly struct StrikeEventData
{
    public readonly CharacterType Source;
    public readonly float Damage;
    public readonly Vector2 Position;
    public readonly int ComboStep;
    public readonly DamageType DamageType;
}
```

### DD-2: One file per struct group, not per struct
- `CombatEventData.cs` — all 13 combat event structs (3-5 fields each, semantically related)
- `RunEventData.cs` — all 7 run event structs
- Enums stay one-file-per-enum (8 files) to minimize merge conflicts across 3 devs

### DD-3: Forward-declare `AttackData` and `PathData` as empty SO stubs
`IAttacker.CurrentAttack` returns `AttackData` (defined in T005) and `IPathProvider.MainPath` returns `PathData` (defined in T008). Declare both as empty ScriptableObject stubs in `Shared/Data/` so interfaces compile now. Real fields added in their respective tasks.

```csharp
// Shared/Data/AttackData.cs — stub, filled in T005
[CreateAssetMenu(menuName = "TomatoFighters/AttackData")]
public class AttackData : ScriptableObject { }

// Shared/Data/PathData.cs — stub, filled in T008
[CreateAssetMenu(menuName = "TomatoFighters/PathData")]
public class PathData : ScriptableObject { }
```

### DD-4: Event data field definitions

| Event Struct | Fields | Reasoning |
|-------------|--------|-----------|
| `StrikeEventData` | source, damage, position, comboStep, damageType | Roguelite needs damage type for ritual triggers |
| `SkillEventData` | source, damage, position, damageType | Heavy/skill attack data |
| `ArcanaEventData` | source, damage, position, damageType, manaCost | Mana tracking for Mystica synergies |
| `DashEventData` | source, startPosition, endPosition | Dash distance matters for Viper's Distance Bonus |
| `DeflectEventData` | source, attacker, timing (early/perfect/late) | Timing affects ritual bonuses |
| `ClashEventData` | source, opponent, result (won/lost/draw) | Different triggers for win vs lose |
| `PunishEventData` | source, target, damage, wasFinisher | Finisher triggers separate from normal punish |
| `KillEventData` | source, enemyType, overkillAmount, position | Overkill could trigger ritual effects |
| `FinisherEventData` | source, target, damage, position | Separate from Kill for ritual tracking |
| `JumpEventData` | source, position | Simple — jump tracking for ritual triggers |
| `DodgeEventData` | source, position, dodgedAttackType | What was dodged matters for counter-triggers |
| `TakeDamageEventData` | target, damage, source, wasBlocked, remainingHP | HP thresholds trigger path abilities |
| `PathAbilityEventData` | source, abilityId, pathType, tier | Track which path ability was used |

### DD-5: `PathData` forward-declare pattern
Same as AttackData — `IPathProvider` returns `PathData` which is defined in T008. Declare as empty SO stub here.

## Implementation Notes

- **Namespace convention:** All files use `TomatoFighters.Shared.Interfaces`, `TomatoFighters.Shared.Enums`, `TomatoFighters.Shared.Data`
- **No MonoBehaviour dependencies** — interfaces and data structs are pure C#. Only exception: `DamagePacket` and event structs use `Vector2` from `UnityEngine`
- **XML doc comments on every public member** — these interfaces are the project's contract; clarity is non-negotiable
- **Enums: one file per enum** — separate files keep merges clean when all 3 devs touch Shared/
- **Forward-declare types** — `AttackData`, `PathData`, `OnHitEffect`, `OnTriggerEffect`, `PathAbility` are stubs here, filled in by later tasks

## Acceptance Criteria

- [ ] ICombatEvents with all 13 event signatures (12 original + OnPathAbilityUsed)
- [ ] IBuffProvider with all 10 methods including path-extended queries
- [ ] IPathProvider with Character property, tier queries, and ability unlock check
- [ ] IDamageable with TakeDamage, health properties, pressure, knockback, launch, invulnerability
- [ ] IAttacker with CurrentAttack, telegraph, and punish state
- [ ] IRunProgressionEvents with all 9 events including path selection and tier-up
- [ ] All 8 shared enums defined with correct values per design spec
- [ ] DamagePacket struct defined with all 6 fields
- [ ] All 13 combat event data structs defined
- [ ] All 7 run progression event data structs defined
- [ ] Placeholder types for OnHitEffect, OnTriggerEffect, PathAbility exist
- [ ] Every public member has XML doc comment
- [ ] Compiles with zero warnings
- [ ] No references to Combat/, Roguelite/, or World/ namespaces

## References

- `architecture/interface-contracts.md` — Canonical interface definitions
- `architecture/data-flow.md` — How data flows through the interfaces per frame
- `architecture/system-overview.md` — 3-pillar module map
- `design-specs/CHARACTER-ARCHETYPES.md` — Character stats, paths, stat types
- `developer/coding-standards.md` — Naming conventions, architecture rules
