# T006: CharacterBaseStats ScriptableObject

## Metadata
| Field | Value |
|-------|-------|
| **Phase** | 1 — Foundation |
| **Type** | implementation |
| **Priority** | P0 |
| **Owner** | Dev 2 |
| **Agent** | so-architect |
| **Depends On** | T001 |
| **Blocks** | T007, T008 |
| **Status** | PENDING |
| **Branch** | `pillar2/T006-character-base-stats` |

## Objective
Create the `CharacterBaseStats` ScriptableObject class defining 8 base stats per character, and produce all 4 character stat assets with values from CHARACTER-ARCHETYPES.md. This is the foundation all stat calculations, path bonuses, and balance tuning build upon.

## Context
Every character in Tomato Fighters is differentiated by base stats from the moment the run begins. The 8-stat framework (HP, DEF, ATK, SPD, MNA, MRG, CRT, PRS) is shared across all characters but with dramatically different starting values per archetype. These stats feed into `CharacterStatCalculator` (T007), which layers path bonuses, ritual multipliers, trinket modifiers, and soul tree bonuses on top.

The `CharacterBaseStats` SO is a pure data container — no logic. It lives in `Shared/Data/` because all 3 pillars (Combat, Roguelite, World) need to read character stats through `IPathProvider` and `IBuffProvider`.

The project follows a strict no-singletons, ScriptableObject-for-data architecture. All file paths are relative to `Assets/Scripts/`.

## Requirements
1. Create `CharacterBaseStats.cs` ScriptableObject class in `Shared/Data/`
2. Define 8 stat fields matching the shared stat framework:
   - `health` (HP) — int, range 50-200
   - `defense` (DEF) — int, range 0-30
   - `attack` (ATK) — float, range 0.5-2.0
   - `speed` (SPD) — float, range 0.7-1.5
   - `mana` (MNA) — int, range 30-150
   - `manaRegen` (MRG) — float, range 1-8
   - `critChance` (CRT) — float, range 0.0-0.25 (stored as decimal, not percentage)
   - `stunRate` (PRS) — float, range 0.5-2.0 — maps to `StatType.StunRate` from T001
3. Include a `CharacterType` enum reference field (Brutor, Slasher, Mystica, Viper)
4. Include a `passiveAbilityId` string field identifying the character's passive (e.g., `"ThickSkin"`, `"Bloodlust"`, `"ArcaneResonance"`, `"DistanceBonus"`)
5. Include a `[TextArea]` description field for editor reference
6. Add `[CreateAssetMenu]` attribute for easy asset creation in Unity Editor
7. Add `[Header]` and `[Tooltip]` attributes for stat groups to aid designer workflow
8. Include `rangedAttack` as a first-class stat field on ALL characters (not Viper-only). Used by throwables and ranged moves. No sentinel value — every character has a real ranged attack multiplier. `attack` = melee only.
9. Create 4 character stat assets in `ScriptableObjects/Characters/`:

### Stat Values (from CHARACTER-ARCHETYPES.md)

| Stat | Brutor | Slasher | Mystica | Viper |
|------|--------|---------|---------|-------|
| HP | 200 | 100 | 50 | 80 |
| DEF | 25 | 8 | 5 | 10 |
| ATK (melee) | 0.7 | 2.0 | 0.5 | 0.6 |
| ATK (ranged) | 0.5 | 0.8 | 0.6 | 1.8 |
| SPD | 0.7 | 1.3 | 1.0 | 1.1 |
| MNA | 50 | 60 | 150 | 120 |
| MRG | 2 | 3 | 8 | 6 |
| CRT | 0.05 | 0.15 | 0.05 | 0.10 |
| STN (stunRate) | 1.0 | 1.5 | 0.5 | 0.8 |
| Passive | ThickSkin | Bloodlust | ArcaneResonance | DistanceBonus |

## File Plan
| File | Description |
|------|-------------|
| `Assets/Scripts/Shared/Data/CharacterBaseStats.cs` | ScriptableObject class with 8 stat fields, passive ID, CharacterType, CreateAssetMenu |
| `Assets/ScriptableObjects/Characters/BrutorStats.asset` | Brutor stat values (tank archetype) |
| `Assets/ScriptableObjects/Characters/SlasherStats.asset` | Slasher stat values (melee DPS archetype) |
| `Assets/ScriptableObjects/Characters/MysticaStats.asset` | Mystica stat values (support/mage archetype) |
| `Assets/ScriptableObjects/Characters/ViperStats.asset` | Viper stat values (ranged DPS archetype) |

## Implementation Notes
- **No singletons** — this is a pure data SO, referenced via SerializeField injection
- **CRT stored as decimal** — 0.05 means 5%, not 5. All runtime calculations use decimal form
- **`attack` vs `rangedAttack`** — `attack` is melee-only. `rangedAttack` is ranged/throwable-only. All 4 characters have both as real values — no sentinel. This future-proofs throwable items that any character can equip.
- **Stat ranges are design constraints, not runtime validation** — use `[Range]` attributes for editor guidance but don't clamp at runtime (path bonuses can push stats beyond base ranges)
- **CharacterType enum** should already be defined in T001 (`Shared/Enums/`); reference it, don't redefine it
- **Asset creation**: `.asset` files are created in Unity Editor via `Create > Tomato Fighters > Character Base Stats` (from CreateAssetMenu). The agent should create the C# class; asset creation requires Unity Editor (mark with implementation notes if CLI-only)

## Acceptance Criteria
- [ ] `CharacterBaseStats` ScriptableObject class with all 8 stat fields
- [ ] `passiveAbilityId` string field present
- [ ] `CharacterType` enum field present
- [ ] `[CreateAssetMenu]` attribute configured
- [ ] 4 character stat assets created with correct values
- [ ] Stat values match CHARACTER-ARCHETYPES.md exactly (see table above)
- [ ] Compiles with zero warnings
- [ ] No singleton patterns used

## Design Decisions

### DD-1: `stunRate` as canonical name for pressure rate stat
**Decision:** Use `stunRate` (field) / `StatType.StunRate` (enum) as the canonical C# identifier for the stat that controls how fast a character fills enemy pressure meters. Do NOT use `pressureRate` or `PressureRate` anywhere in code.

**Rationale:** T001 already shipped with `StatType.StunRate` in the enum. Renaming it would have zero benefit at this stage — nothing has consumed it yet, but keeping a single canonical name avoids enum duplication. All downstream tasks (T007, T008, T026 PressureSystem) must reference `StatType.StunRate`.

**Impact on docs:** Design documents (CHARACTER-ARCHETYPES.md, CLAUDE.md) still use the abbreviation "PRS" and the label "Pressure Rate" for readability. These are human-facing labels only. Code always uses `stunRate`/`StunRate`.

```csharp
// CORRECT
[Range(0.5f, 2.0f)]
public float stunRate = 1.0f;   // StatType.StunRate

// WRONG — never use this
public float pressureRate;
```

### DD-2: `rangedAttack` is a first-class stat on all characters
**Decision:** Every character has both `attack` (melee) and `rangedAttack` fields with real values. No sentinel (-1). `StatType.RangedAttack` (already in T001 enum) maps to `rangedAttack`.

**Rationale:** Throwables will be available to all characters in a future task. If `rangedAttack` were Viper-only with a sentinel, adding throwables would require a schema migration on all 4 assets. With a first-class field on everyone, throwable items simply use `rangedAttack` and it already works.

**Base rangedAttack values (non-specialist defaults):**
- Brutor: 0.5 — heavy, not built for throwing
- Slasher: 0.8 — can throw knives, competent but not his focus
- Mystica: 0.6 — awkward with physical throwables, her ranged is magical
- Viper: 1.8 — ranged specialist, unchanged

```csharp
[Header("Attack")]
[Range(0.1f, 2.0f)]
[Tooltip("Melee damage multiplier.")]
public float attack = 1.0f;

[Range(0.1f, 2.0f)]
[Tooltip("Ranged/throwable damage multiplier. Used by throwable items for all characters.")]
public float rangedAttack = 0.5f;
```

### DD-3: `[CreateAssetMenu]` path uses `"TomatoFighters/Data/"` grouping
**Decision:** `[CreateAssetMenu(menuName = "TomatoFighters/Data/Character Base Stats", fileName = "NewCharacterBaseStats")]`

**Rationale:** As AttackData, PathData, RitualData, TrinketData, and more SOs are added in later tasks, a flat `TomatoFighters/` menu becomes unwieldy. The `Data/` subfolder mirrors the `Assets/Scripts/Shared/Data/` folder structure and makes the Create menu scannable.

**Convention for all future SOs in `Shared/Data/`:** use `"TomatoFighters/Data/<Name>"`.

## References
- [CHARACTER-ARCHETYPES.md](/design-specs/CHARACTER-ARCHETYPES.md) — Stat values (Stat Comparison table, lines 500-510), passive descriptions
- [TASK_BOARD.md](/TASK_BOARD.md) — T006 entry
- T001 (dependency) — Shared enums including `CharacterType`, `StatType`
