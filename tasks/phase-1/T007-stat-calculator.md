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
| **Status** | DONE |
| **Completed** | 2026-03-02 |
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
2. Define a `FinalStats` struct containing all 8 calculated stats:
   - `health` (int)
   - `defense` (int)
   - `attack` (float)
   - `speed` (float)
   - `mana` (int)
   - `manaRegen` (float)
   - `critChance` (float) — clamped to 0.0-1.0
   - `pressureRate` (float)
3. For Viper, `FinalStats` should include both `meleeAttack` and `rangedAttack` (or a method to query attack by type)
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
| `Assets/Scripts/Paths/CharacterStatCalculator.cs` | Pure C# calculator class with `Calculate()` and `CalculateSingleStat()` methods |
| `Assets/Scripts/Paths/FinalStats.cs` | (Optional, can be nested) Struct holding all 8 final stat values |
| `Assets/Scripts/Paths/StatModifierInput.cs` | (Optional, can be nested) Input struct bundling all modifier sources |

## Implementation Notes
- **Pure C# for testability** — this is a deliberate architecture decision. The calculator should have zero Unity dependencies beyond reading from CharacterBaseStats (which is a ScriptableObject but can be mocked in tests)
- **No caching** — recalculate every time. Stats change frequently during a run (ritual pickups, trinket equips, path tier-ups). Caching adds complexity for negligible performance gain on 8 float multiplications
- **StatType enum** from T001 should be used as array indices: `Health=0, Defense=1, Attack=2, Speed=3, Mana=4, ManaRegen=5, CritChance=6, PressureRate=7`
- **Path bonuses are additive, everything else is multiplicative** — this is the core balance lever. Adding +20 HP is predictable; multiplying by 1.5x from rituals creates exponential scaling that makes late-game feel powerful
- **Viper's split ATK**: when calculating attack for Viper, the calculator needs to know whether to use `meleeAttack` or `rangedAttack` base. Consider a `bool isRanged` parameter or always calculate both
- **Thread safety**: the calculator is stateless (all inputs via parameters), so it's inherently thread-safe
- **Unit test examples**:
  - Brutor base stats with no modifiers = raw base values
  - Slasher with T1 Executioner path (+0.3 ATK, +0.05 CRT) = base + path
  - Mystica with path bonuses + 1.5x ritual multiplier = (base + path) * 1.5
  - All multipliers at 1.0 = base stats unchanged

## Acceptance Criteria
- [ ] Pure C# class, **not** a MonoBehaviour or ScriptableObject
- [ ] Correct formula: `(Base + PathBonuses) * Ritual * Trinket * SoulTree` applied per stat independently
- [ ] Returns `FinalStats` struct with all 8 stats
- [ ] Integer stats (HP, DEF, MNA) rounded after multiplication
- [ ] CritChance clamped to 0.0-1.0
- [ ] Default multipliers of 1.0 produce unmodified base stats
- [ ] Handles Viper's split ATK (melee vs ranged)
- [ ] Unit testable with mock/constructed inputs (no scene required)
- [ ] Compiles with zero warnings

## References
- [CHARACTER-ARCHETYPES.md](/design-specs/CHARACTER-ARCHETYPES.md) — Stat Growth section (lines 56-61), formula description
- [TASK_BOARD.md](/TASK_BOARD.md) — T007 entry
- T006 (dependency) — `CharacterBaseStats` ScriptableObject providing base values
- T017 (blocked) — Character Passives read from this calculator
- T025 (blocked) — HUD displays stats from this calculator
