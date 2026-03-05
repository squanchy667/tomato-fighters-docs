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
| **Status** | DONE |
| **Completed** | 2026-03-02 |
| **Branch** | `shared/T001-shared-contracts` |

## Objective
Define every cross-pillar interface contract, shared enum, and shared data structure that the 3-pillar architecture depends on. This is the foundational layer that enables all three developers to work independently on Combat, Roguelite, and World pillars without importing from each other's namespaces.

## Context
Tomato Fighters uses a strict 3-pillar architecture (Combat / Roguelite / World) where **all cross-domain communication flows through `Shared/Interfaces/`**. No pillar may import from another pillar's namespace — ever. This task produces the contracts that every subsequent task depends on. It is a sync session: all 3 devs must agree on every interface signature before implementation proceeds.

Unity project root: `unity/TomatoFighters/`
All script paths below are relative to `unity/TomatoFighters/Assets/`.

Reference the full interface specs in `architecture/interface-contracts.md` and the data flow diagrams in `architecture/data-flow.md`. Character stats and path enums come from `design-specs/CHARACTER-ARCHETYPES.md`.

---

## Requirements

### Interfaces (6 files)

#### 1. ICombatEvents (`Shared/Interfaces/ICombatEvents.cs`)
Combat fires, Roguelite subscribes. 13 event signatures using `Action<T>`:

```csharp
public interface ICombatEvents
{
    event Action<StrikeEventData> OnStrike;
    event Action<SkillEventData> OnSkill;
    event Action<ArcanaEventData> OnArcana;
    event Action<DashEventData> OnDash;
    event Action<DeflectEventData> OnDeflect;
    event Action<ClashEventData> OnClash;
    event Action<PunishEventData> OnPunish;
    event Action<KillEventData> OnKill;
    event Action<FinisherEventData> OnFinisher;
    event Action<JumpEventData> OnJump;
    event Action<DodgeEventData> OnDodge;
    event Action<TakeDamageEventData> OnTakeDamage;
    event Action<PathAbilityEventData> OnPathAbilityUsed;
}
```

#### 2. IBuffProvider (`Shared/Interfaces/IBuffProvider.cs`)
Roguelite provides, Combat queries. 10 methods:

```csharp
public interface IBuffProvider
{
    float GetDamageMultiplier(DamageType type);
    float GetSpeedMultiplier();
    float GetDefenseMultiplier();
    List<OnHitEffect> GetAdditionalOnHitEffects();
    List<OnTriggerEffect> GetTriggerEffects(RitualTrigger trigger);
    bool IsRepetitivePenaltyOverridden();
    float GetPathDamageMultiplier();
    float GetPathDefenseMultiplier();
    float GetPathSpeedMultiplier();
    List<PathAbility> GetActivePathAbilities();
}
```

Default return values when no buffs active: `1.0f` for multipliers, empty lists, `false` for overrides.

#### 3. IPathProvider (`Shared/Interfaces/IPathProvider.cs`)
Roguelite provides, Combat + World query. 8 members:

```csharp
public interface IPathProvider
{
    CharacterType Character { get; }
    object MainPath { get; }           // TODO: Replace with PathData when T008 lands
    object SecondaryPath { get; }      // TODO: Replace with PathData when T008 lands
    int MainPathTier { get; }          // 1-3, 0 if not selected
    int SecondaryPathTier { get; }     // 1-2, 0 if not selected
    bool HasPath(PathType type);
    float GetPathStatBonus(StatType stat);
    bool IsPathAbilityUnlocked(string abilityId);
}
```

> **Forward reference:** `MainPath`/`SecondaryPath` return `object` because the real `PathData` ScriptableObject is defined in T008. Update to `PathData` when T008 lands.

#### 4. IDamageable (`Shared/Interfaces/IDamageable.cs`)
Combat defines, World implements on enemies. 8 members:

```csharp
public interface IDamageable
{
    void TakeDamage(DamagePacket damage);
    float CurrentHealth { get; }
    float MaxHealth { get; }
    void AddStun(float amount);        // Fill hidden stun meter
    bool IsStunned { get; }
    void ApplyKnockback(Vector2 force);
    void ApplyLaunch(Vector2 force);
    bool IsInvulnerable { get; }
}
```

> **Renamed:** `AddPressure` → `AddStun` to match the "stun" terminology.

#### 5. IAttacker (`Shared/Interfaces/IAttacker.cs`)
World implements on enemies, Combat reads. 5 members:

```csharp
public interface IAttacker
{
    object CurrentAttack { get; }      // TODO: Replace with AttackData when T005 lands
    bool IsCurrentAttackUnstoppable { get; }
    TelegraphType CurrentTelegraphType { get; }
    float PunishWindowDuration { get; }
    bool IsInPunishableState { get; }
}
```

> **Forward reference:** `CurrentAttack` returns `object` because the real `AttackData` ScriptableObject is defined in T005. It provides the attack data for the currently executing enemy attack — Combat reads it to check telegraph type and punish state when the player tries to deflect/clash. Update to `AttackData` when T005 lands.

#### 6. IRunProgressionEvents (`Shared/Interfaces/IRunProgressionEvents.cs`)
World fires, Roguelite subscribes. 9 events:

```csharp
public interface IRunProgressionEvents
{
    event Action<AreaClearedData> OnAreaCleared;
    event Action<BossDefeatedData> OnBossDefeated;
    event Action<IslandCompletedData> OnIslandCompleted;
    event Action<ShopEnteredData> OnShopEntered;
    event Action OnRunStarted;
    event Action<RunEndData> OnRunEnded;
    event Action<PathSelectedData> OnMainPathSelected;
    event Action<PathSelectedData> OnSecondaryPathSelected;
    event Action<PathTierUpData> OnPathTierUp;
}
```

---

### Enums (9 files, one per enum)

| # | File | Enum | Values | Count |
|---|------|------|--------|-------|
| 1 | `Shared/Enums/CharacterType.cs` | `CharacterType` | Brutor, Slasher, Mystica, Viper | 4 |
| 2 | `Shared/Enums/PathType.cs` | `PathType` | Warden, Bulwark, Guardian, Executioner, Reaper, Shadow, Sage, Enchanter, Conjurer, Marksman, Trapper, Arcanist | 12 |
| 3 | `Shared/Enums/StatType.cs` | `StatType` | Health, Defense, Attack, RangedAttack, Speed, Mana, ManaRegen, CritChance, StunRate, CancelWindow | **10** |
| 4 | `Shared/Enums/DamageType.cs` | `DamageType` | Physical, Fire, Lightning, Water, Thorn, Gale, Time, Cosmic, Necro | 9 |
| 5 | `Shared/Enums/RitualTrigger.cs` | `RitualTrigger` | OnStrike, OnSkill, OnDash, OnDeflect, OnClash, OnFinisher, OnKill, OnArcana, OnJump, OnDodge, OnTakeDamage | 11 |
| 6 | `Shared/Enums/TelegraphType.cs` | `TelegraphType` | Normal, Unstoppable | 2 |
| 7 | `Shared/Enums/RitualFamily.cs` | `RitualFamily` | Fire, Lightning, Water, Thorn, Gale, Time, Cosmic, Necro | 8 |
| 8 | `Shared/Enums/RitualCategory.cs` | `RitualCategory` | Core, General, Enhancement, Twin | 4 |
| 9 | `Shared/Enums/DamageResponse.cs` | `DamageResponse` | Hit, Deflected, Clashed, Dodged | 4 |

> **StatType changes from original spec:**
> - `PressureRate` → `StunRate` (renamed to match stun terminology)
> - `CancelWindow` added (clash/deflect cancel window length)
> - Total: 10 values (was 9)
>
> **DamageResponse** added — required by `TakeDamageEventData.response` field.

---

### Data Structures (4 files)

#### DamagePacket (`Shared/Data/DamagePacket.cs`)

Immutable payload passed to `IDamageable.TakeDamage`. Contains everything the target needs to process one hit.

```csharp
public readonly struct DamagePacket
{
    public readonly DamageType type;           // Physical, Fire, etc.
    public readonly float amount;              // Final calculated damage
    public readonly bool isPunishDamage;       // 2x stun fill
    public readonly Vector2 knockbackForce;    // Applied to Rigidbody2D
    public readonly Vector2 launchForce;       // Applied to Rigidbody2D
    public readonly CharacterType source;      // Who dealt it (for passive tracking)
}
```

#### Combat Event Data Structs (`Shared/Data/CombatEventData.cs`)

13 `readonly struct` types — one per `ICombatEvents` event. GC-friendly for combat frame usage.

| Struct | Fields |
|--------|--------|
| `StrikeEventData` | character, damage, damageType, comboIndex, hitPosition |
| `SkillEventData` | character, damage, damageType, hitPosition |
| `ArcanaEventData` | character, arcanaId, damage, damageType, manaCost |
| `DashEventData` | character, dashDirection, hasIFrames |
| `DeflectEventData` | character, incomingDamage, incomingType |
| `ClashEventData` | character, incomingDamage, incomingType |
| `PunishEventData` | character, damage, damageType, hitPosition |
| `KillEventData` | character, killPosition, killingDamageType |
| `FinisherEventData` | character, damage, damageType, comboLength, hitPosition |
| `JumpEventData` | character, isAirborne |
| `DodgeEventData` | character, dodgeDirection |
| `TakeDamageEventData` | character, damageAmount, damageType, response (DamageResponse) |
| `PathAbilityEventData` | character, abilityId, pathType, tier, manaCost |

> Keep event data minimal — only fields that the subscriber (RitualSystem) genuinely needs.

#### Run Event Data Structs (`Shared/Data/RunEventData.cs`)

7 `readonly struct` types — one per `IRunProgressionEvents` event (except `OnRunStarted` which has no data).

| Struct | Fields |
|--------|--------|
| `AreaClearedData` | areaIndex, enemiesDefeated |
| `BossDefeatedData` | bossId, islandIndex |
| `IslandCompletedData` | islandIndex |
| `ShopEnteredData` | islandIndex, isSpecialShop |
| `RunEndData` | wasVictory, crystalsEarned, finalIslandIndex |
| `PathSelectedData` | character, pathType, isMainPath |
| `PathTierUpData` | character, pathType, newTier |

#### Placeholder Types (`Shared/Data/PlaceholderTypes.cs`)

Forward-declared types referenced by `IBuffProvider`. Will be fleshed out in later tasks.

| Type | Purpose | Fleshed out in |
|------|---------|----------------|
| `OnHitEffect` | Ritual on-hit effects (burn, chain lightning, etc.) | T021 (RitualSystem) |
| `OnTriggerEffect` | Trigger-based effects from rituals | T021 (RitualSystem) |
| `PathAbility` | Path ability data for combat execution | T028 (PathAbilityExecutor) |

---

### Assembly Definitions (4 files)

Enforce pillar boundaries at **compile time** — a pillar that tries to import another pillar will fail to compile.

| File | Assembly Name | References |
|------|---------------|------------|
| `Scripts/Shared/TomatoFighters.Shared.asmdef` | TomatoFighters.Shared | Unity only |
| `Scripts/Combat/TomatoFighters.Combat.asmdef` | TomatoFighters.Combat | TomatoFighters.Shared only |
| `Scripts/Roguelite/TomatoFighters.Roguelite.asmdef` | TomatoFighters.Roguelite | TomatoFighters.Shared only |
| `Scripts/World/TomatoFighters.World.asmdef` | TomatoFighters.World | TomatoFighters.Shared only |

> **Characters/** and **Paths/** share the assembly of their parent pillar (Combat and Roguelite respectively).

---

## Directory Structure

All paths relative to `unity/TomatoFighters/Assets/`:

```
Scripts/
  Shared/
    TomatoFighters.Shared.asmdef
    Interfaces/
      ICombatEvents.cs
      IBuffProvider.cs
      IPathProvider.cs
      IDamageable.cs
      IAttacker.cs
      IRunProgressionEvents.cs
    Enums/
      CharacterType.cs
      PathType.cs
      StatType.cs
      DamageType.cs
      RitualTrigger.cs
      TelegraphType.cs
      RitualFamily.cs
      RitualCategory.cs
      DamageResponse.cs
    Data/
      DamagePacket.cs
      CombatEventData.cs
      RunEventData.cs
      PlaceholderTypes.cs
  Combat/
    TomatoFighters.Combat.asmdef
  Characters/                          (empty — future tasks)
  Roguelite/
    TomatoFighters.Roguelite.asmdef
  Paths/                               (empty — future tasks)
  World/
    TomatoFighters.World.asmdef

ScriptableObjects/                     (empty scaffold directories)
  Attacks/
  Characters/
  Enemies/
  Inspirations/
  Islands/
  Paths/
  Rituals/
  Trinkets/
```

**Total files created:** 22 (6 interfaces + 9 enums + 4 data + 4 asmdef) = **23 files**

---

## File Plan

| # | File Path | Description |
|---|-----------|-------------|
| 1 | `Scripts/Shared/TomatoFighters.Shared.asmdef` | Assembly definition — references Unity only |
| 2 | `Scripts/Shared/Enums/CharacterType.cs` | Character enum (4 values) |
| 3 | `Scripts/Shared/Enums/PathType.cs` | Path enum (12 values, grouped by character) |
| 4 | `Scripts/Shared/Enums/StatType.cs` | Stat enum (10 values) |
| 5 | `Scripts/Shared/Enums/DamageType.cs` | Damage type enum (9 values) |
| 6 | `Scripts/Shared/Enums/RitualTrigger.cs` | Ritual trigger enum (11 values) |
| 7 | `Scripts/Shared/Enums/TelegraphType.cs` | Telegraph enum (2 values) |
| 8 | `Scripts/Shared/Enums/RitualFamily.cs` | Ritual family enum (8 values) |
| 9 | `Scripts/Shared/Enums/RitualCategory.cs` | Ritual category enum (4 values) |
| 10 | `Scripts/Shared/Enums/DamageResponse.cs` | Damage response enum (4 values) |
| 11 | `Scripts/Shared/Data/DamagePacket.cs` | Damage payload readonly struct |
| 12 | `Scripts/Shared/Data/CombatEventData.cs` | 13 combat event data readonly structs |
| 13 | `Scripts/Shared/Data/RunEventData.cs` | 7 run progression event data readonly structs |
| 14 | `Scripts/Shared/Data/PlaceholderTypes.cs` | OnHitEffect, OnTriggerEffect, PathAbility stubs |
| 15 | `Scripts/Shared/Interfaces/ICombatEvents.cs` | 13-event combat-to-roguelite interface |
| 16 | `Scripts/Shared/Interfaces/IBuffProvider.cs` | 10-method buff query interface |
| 17 | `Scripts/Shared/Interfaces/IPathProvider.cs` | 8-member path state interface |
| 18 | `Scripts/Shared/Interfaces/IDamageable.cs` | 8-member damage/health/stun interface |
| 19 | `Scripts/Shared/Interfaces/IAttacker.cs` | 5-member attack state interface |
| 20 | `Scripts/Shared/Interfaces/IRunProgressionEvents.cs` | 9-event run progression interface |
| 21 | `Scripts/Combat/TomatoFighters.Combat.asmdef` | Assembly definition — references Shared |
| 22 | `Scripts/Roguelite/TomatoFighters.Roguelite.asmdef` | Assembly definition — references Shared |
| 23 | `Scripts/World/TomatoFighters.World.asmdef` | Assembly definition — references Shared |

---

## Creation Order

Strict dependency order:

1. **Directory structure** — all folders under `Scripts/` and `ScriptableObjects/`
2. **Assembly definitions** (4 files) — no code dependencies
3. **Enums** (9 files) — no code dependencies
4. **Data structures** (4 files) — depend on enums + `UnityEngine.Vector2`
5. **Interfaces** (6 files) — depend on enums + data structs

```
Enums (9 files) ──────────────────┐
                                  ├──► Data Structs (4 files) ──► Interfaces (6 files)
Assembly Defs (4 files) ──────────┘
```

---

## Implementation Notes

### Namespaces
- `TomatoFighters.Shared.Interfaces` — all 6 interfaces
- `TomatoFighters.Shared.Enums` — all 9 enums
- `TomatoFighters.Shared.Data` — all 4 data/struct files

### Rules
- **No MonoBehaviour dependencies** — interfaces and data structs are pure C#. Only exception: `DamagePacket` and event data structs use `Vector2` from `UnityEngine`
- **Event data structs are `readonly struct`** — GC-friendly for combat-frame usage
- **Keep event data minimal** — only fields that subscribers genuinely need
- **XML doc comments on every public member** — `<summary>` tags explaining what, when, and who fires/implements
- **One file per enum** — keeps merges clean when all 3 devs touch Shared/
- **Forward references use `object`** — `IAttacker.CurrentAttack` and `IPathProvider.MainPath`/`SecondaryPath` return `object` until T005/T008 land with the real SO types

### Assembly Definition Details
Each `.asmdef` is a JSON file. Example for Shared:
```json
{
    "name": "TomatoFighters.Shared",
    "rootNamespace": "TomatoFighters.Shared",
    "references": [],
    "includePlatforms": [],
    "excludePlatforms": [],
    "allowUnsafeCode": false,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": true,
    "defineConstraints": [],
    "versionDefines": [],
    "noEngineReferences": false
}
```

Pillar asmdefs reference Shared:
```json
{
    "name": "TomatoFighters.Combat",
    "rootNamespace": "TomatoFighters.Combat",
    "references": ["TomatoFighters.Shared"],
    ...
}
```

---

## Acceptance Criteria

- [ ] `ICombatEvents` with all 13 event signatures (12 original + `OnPathAbilityUsed`)
- [ ] `IBuffProvider` with all 10 methods including path-extended queries
- [ ] `IPathProvider` with Character property, tier queries, and ability unlock check (8 members)
- [ ] `IDamageable` with TakeDamage, health properties, stun, knockback, launch, invulnerability (8 members)
- [ ] `IAttacker` with CurrentAttack, telegraph, and punish state (5 members)
- [ ] `IRunProgressionEvents` with all 9 events including path selection and tier-up
- [ ] All 9 shared enums defined with correct values per spec
- [ ] `StatType` has 10 values including `StunRate` and `CancelWindow`
- [ ] `DamageResponse` enum exists with 4 values
- [ ] `DamagePacket` readonly struct with all 6 fields
- [ ] All 13 combat event data readonly structs defined
- [ ] All 7 run progression event data readonly structs defined
- [ ] Placeholder types for `OnHitEffect`, `OnTriggerEffect`, `PathAbility` exist
- [ ] 4 assembly definitions enforce pillar boundaries
- [ ] Every public member has `<summary>` XML doc comment
- [ ] Compiles with zero errors and zero warnings
- [ ] No `using` references to `Combat/`, `Roguelite/`, or `World/` namespaces

---

## Post-T001 Unblocked Tasks

Once T001 is complete, these tasks become unblocked and can be worked in parallel:

| Task | Description | Owner |
|------|-------------|-------|
| T002 | CharacterController2D | Dev 1 |
| T003 | InputBufferSystem | Dev 1 |
| T005 | AttackData SO | Dev 1 |
| T006 | CharacterBaseStats SO | Dev 2 |
| T008 | PathData SO | Dev 2 |
| T009 | CurrencyManager | Dev 2 |
| T010 | WaveManager | Dev 3 |
| T011 | EnemyBase | Dev 3 |
| T012 | CameraController2D | Dev 3 |

---

## Data Flow Reference

```
Player Input
    │
    ▼
[Combat Pillar]
    │── CharacterController2D → movement/dash
    │── InputBuffer → ComboSystem → attack
    │── HitboxManager → collision check
    │── DefenseSystem → deflect/clash/dodge
    │
    ├── FIRES ICombatEvents ──────────────────► [Roguelite Pillar]
    │                                               │── RitualSystem triggers
    │                                               │── Stacking calculation
    │                                               │── Buff values updated
    │                                               │
    ◄── QUERIES IBuffProvider ◄──────────────────── │
    │   (damage × ritual × path × trinket)
    │
    │── CALLS IDamageable.TakeDamage ────────► [World Pillar]
    │                                               │── EnemyBase takes damage
    │                                               │── EnemyAI state change
    │                                               │── BossAI phase check
    │                                               │
    │                                               ├── FIRES IRunProgressionEvents
    │                                               │       │
    │                                               │       ▼
    │                                               │   [Roguelite Pillar]
    │                                               │   │── Reward selection
    │                                               │   │── Path tier-up
    │                                               │   └── Currency update
    │
    ▼
[Frame rendered with HUD updates from World Pillar]
```

---

## References

- `architecture/interface-contracts.md` — Canonical interface definitions
- `architecture/data-flow.md` — How data flows through the interfaces per frame
- `architecture/system-overview.md` — 3-pillar module map
- `design-specs/CHARACTER-ARCHETYPES.md` — Character stats, paths, stat types
- `developer/coding-standards.md` — Naming conventions, architecture rules
