# T007: CharacterStatCalculator

## Metadata
| Field | Value |
|-------|-------|
| **Phase** | 1 — Foundation |
| **Type** | implementation |
| **Priority** | P1 |
| **Owner** | Dev 2 |
| **Agent** | roguelite-agent |
| **Depends On** | T006 |
| **Blocks** | T017, T025 |
| **Status** | PENDING |
| **Branch** | `pillar2/T007-stat-calculator` |

## Objective
Create a pure C# class that calculates final character stats by layering path bonuses, ritual multipliers, trinket modifiers, and soul tree bonuses on top of base stats. This is the single source of truth for "what are this character's current stats?" at any point during a run.

## Context
The stat growth pipeline in Tomato Fighters is:

```
FinalStat = (BaseStat + Sum(PathBonuses)) * RitualMultiplier * TrinketMultiplier * SoulTreeBonus
```

Each stat is calculated independently — there are no cross-stat dependencies (e.g., HP doesn't affect DEF). This keeps the calculator simple and testable.

The calculator must be a **plain C# class** (not MonoBehaviour) so it can be instantiated in unit tests without a Unity scene. It reads from `CharacterBaseStats` SO (T006) for base values, and accepts path bonuses, ritual multipliers, trinket multipliers, and soul tree bonuses as input parameters.

This class is consumed by combat systems (for damage calculation), UI (for stat display), and the roguelite layer (for checking buff effects). All 3 pillars read through it, so it lives in the `Paths/` directory as part of the shared stat progression system.

## Requirements
1. Create `CharacterStatCalculator.cs` as a plain C# class (no MonoBehaviour, no ScriptableObject)
2. Define a `FinalStats` struct containing all 10 calculated stats:
   - `health` (int)
   - `defense` (int)
   - `attack` (float) — melee
   - `rangedAttack` (float) — **Viper only**; `-1f` when character is not a ranged specialist
   - `throwableAttack` (float) — all characters; scales ground-pickup throwable item damage
   - `speed` (float)
   - `mana` (int)
   - `manaRegen` (float)
   - `critChance` (float) — clamped to 0.0-1.0
   - `stunRate` (float) — design docs call this PRS/Pressure Rate; code uses StunRate (DD-1, T006)
4. Implement the core calculation formula per stat:
   ```
   finalValue = (baseValue + sumOfPathBonuses) * ritualMultiplier * trinketMultiplier * soulTreeBonus
   ```
5. Each multiplier defaults to 1.0 (no modification) when no source is active
6. Accept inputs via method parameters or a `StatModifierInput` struct:
   - `CharacterBaseStats baseStats` — the SO reference
   - `float[] pathBonuses` — per-stat additive bonuses from Main + Secondary paths (indexed by StatType)
   - `float[] ritualMultipliers` — per-stat multiplicative modifiers from active rituals
   - `float[] trinketMultipliers` — per-stat multiplicative modifiers from equipped trinkets
   - `float[] soulTreeBonuses` — per-stat multiplicative modifiers from permanent progression
7. Provide a `Calculate(StatModifierInput input)` method that returns `FinalStats`
8. Provide a `CalculateSingleStat(StatType stat, ...)` method for on-demand single-stat queries
9. Round integer stats (HP, DEF, MNA) to nearest int after all multiplications
10. No clamping on stat values except CRT (capped at 1.0) — path bonuses and multipliers can push stats beyond base ranges

## File Plan
| File | Description |
|------|-------------|
| `Assets/Scripts/Shared/Enums/AttackMode.cs` | Enum: `Melee`, `Ranged`, `Throwable` — used by `FinalStats.GetAttackForMode()` |
| `Assets/Scripts/Paths/FinalStats.cs` | Struct with 10 stat fields + `GetAttackForMode(AttackMode)` helper |
| `Assets/Scripts/Paths/StatModifierInput.cs` | Input struct with float[] arrays + `Default(CharacterBaseStats)` factory |
| `Assets/Scripts/Paths/CharacterStatCalculator.cs` | Pure C# calculator with `Calculate()` and `CalculateSingleStat()` methods |

## Design Decisions

### DD-2: Directory — `Assets/Scripts/Paths/` (not `Roguelite/`)
The calculator is queried by all 3 pillars (Combat for damage math, World for HUD, Roguelite for buff logic). Placing it inside `Roguelite/` would imply pillar ownership. `Paths/` is a neutral shared-ish module that was already listed in the architecture doc. Namespace: `TomatoFighters.Paths`.

### DD-3: 3 separate files
`FinalStats` and `StatModifierInput` are meaningful standalone types read across all pillars. Keeping them in separate files makes them discoverable and keeps `CharacterStatCalculator.cs` focused on logic only.

### DD-4: float[] arrays + `StatModifierInput.Default()` factory
Modifier inputs are `float[]` arrays indexed by `StatType` (11 values). Per-stat multipliers are needed because rituals can target individual stats (e.g. 1.5× ATK only). A static `Default(CharacterBaseStats baseStats)` factory initializes all arrays with correct neutral values (0f for additive path bonuses, 1.0f for all multipliers) so callers don't have to manually construct 44 float slots.

```csharp
// Caller convenience — no modifiers active
var input = StatModifierInput.Default(brutorStats);
FinalStats result = calculator.Calculate(input);
```

`CancelWindow` slot in arrays is silently ignored by the calculator (no base stat on `CharacterBaseStats`). It stays zeroed in `Default()`.

### DD-5: `FinalStats.GetAttackForMode(AttackMode mode)` helper
The `-1f` sentinel for non-Viper `rangedAttack` is an internal data convention that should not leak into Combat or World pillar code. A single helper on `FinalStats` absorbs the sentinel check:

```csharp
public float GetAttackForMode(AttackMode mode) => mode switch
{
    AttackMode.Melee     => attack,
    AttackMode.Ranged    => rangedAttack >= 0f ? rangedAttack : attack,  // non-Viper falls back to melee
    AttackMode.Throwable => throwableAttack,
    _                    => attack
};
```

`AttackMode` is a 3-value enum (`Melee`, `Ranged`, `Throwable`) placed in `Shared/Enums/` since Combat pillar needs it when calling `GetAttackForMode`.

## Implementation Notes
- **Pure C# for testability** — zero Unity dependencies beyond reading from `CharacterBaseStats` (a ScriptableObject, but constructible in tests)
- **No caching** — recalculate every time. Stats change on ritual pickups, trinket equips, path tier-ups. Caching adds complexity for negligible gain on 10 float multiplications
- **StatType enum indices**: `Health=0, Defense=1, Attack=2, RangedAttack=3, ThrowableAttack=4, Speed=5, Mana=6, ManaRegen=7, CritChance=8, StunRate=9, CancelWindow=10`
- **RangedAttack sentinel**: if `baseStats.rangedAttack < 0`, set `FinalStats.rangedAttack = -1f` and skip all modifier math for that slot — never apply multipliers to a sentinel
- **ThrowableAttack**: calculated identically to any other float stat. No special cases. All 4 characters have a positive base value
- **Thread safety**: stateless class — all inputs via parameters, inherently thread-safe
- **Unit test examples**:
  - Brutor `Default()` input → `rangedAttack == -1f`, all other stats match raw base values
  - Slasher T1 Executioner path (`pathBonuses[Attack] = +0.3f, pathBonuses[CritChance] = +0.05f`) → `attack = 2.3f, critChance = 0.2f`
  - Mystica with path bonuses + `ritualMultipliers[Attack] = 1.5f` → `attack = (0.5 + path) * 1.5`
  - Viper `GetAttackForMode(Ranged)` → `1.8f`; Brutor same call → falls back to `0.7f`

## Acceptance Criteria
- [ ] Pure C# class, **not** a MonoBehaviour or ScriptableObject
- [ ] Correct formula: `(Base + PathBonuses) * Ritual * Trinket * SoulTree` applied per stat independently
- [ ] Returns `FinalStats` struct with all 10 stats
- [ ] Integer stats (HP, DEF, MNA) rounded after multiplication
- [ ] CritChance clamped to 0.0–1.0
- [ ] Default multipliers of 1.0 produce unmodified base stats
- [ ] `rangedAttack` is `-1f` for non-Viper with no modifiers applied
- [ ] `throwableAttack` calculated normally for all 4 characters
- [ ] `StatModifierInput.Default(baseStats)` produces neutral (unmodified) output
- [ ] `FinalStats.GetAttackForMode(AttackMode.Ranged)` returns `attack` for non-Viper (no `-1f` leak)
- [ ] `AttackMode` enum in `Shared/Enums/`
- [ ] Unit testable with no scene required
- [ ] Compiles with zero warnings

## References
- [CHARACTER-ARCHETYPES.md](/design-specs/CHARACTER-ARCHETYPES.md) — Stat Growth section (lines 56-61), formula description
- [TASK_BOARD.md](/TASK_BOARD.md) — T007 entry
- T006 (dependency) — `CharacterBaseStats` ScriptableObject providing base values
- T017 (blocked) — Character Passives read from this calculator
- T025 (blocked) — HUD displays stats from this calculator
