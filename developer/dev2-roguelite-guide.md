# Dev 2 Guide — Roguelite Systems

**Owner:** Dev 2
**Pillar:** Roguelite Systems + Character Progression
**Branch prefix:** `pillar2/roguelite-*`
**AgentPilot config:** `unity-roguelite`, `unity-path-system`
**Context strategy:** `roguelite` (sees: `Roguelite/`, `Paths/`, `Shared/`)

---

## Your Domain

You own everything about progression, build-crafting, and the numbers behind the game:

| System | Key Files | What It Does |
|--------|----------|-------------|
| Path System | `Paths/PathSystem.cs`, `PathData.cs` | Main+Secondary selection, tier progression |
| Stat Calculator | `Paths/CharacterStatCalculator.cs` | Base+Path+Ritual+Trinket = final stats |
| Ritual System | `Roguelite/RitualSystem.cs` | 8 families, trigger→effect pipeline |
| Stack Calculator | `Roguelite/RitualStackCalculator.cs` | Multiplicative stacking math |
| Trinket System | `Roguelite/TrinketSystem.cs` | Stat modifier items |
| Inspiration System | `Roguelite/InspirationSystem.cs` | Character-specific move unlocks |
| Meta-Progression | `Roguelite/MetaProgression.cs`, `SoulTree.cs` | Persistent upgrades |
| Currency Manager | `Roguelite/CurrencyManager.cs` | 3 currencies + events |
| Save/Load | `Roguelite/SaveSystem.cs` | JSON persistence |
| Hub Manager | `Roguelite/HubManager.cs` | Between-run base |
| Path Selection UI | `Roguelite/PathSelectionUI.cs` | Upgrade shrine |
| Reward Selection | `Roguelite/RewardSelectorUI.cs` | Post-area picks |

## You Do NOT Touch

- `Combat/` — Dev 1's domain (you PROVIDE buffs via IBuffProvider, never modify combat logic)
- `World/` — Dev 3's domain
- `Characters/` — Dev 1 executes abilities, you DEFINE the data

## How You Communicate with Other Pillars

| Direction | Interface | What Happens |
|-----------|-----------|-------------|
| Dev 1 → You | `ICombatEvents` | Dev 1 fires combat events → your RitualSystem subscribes to trigger effects |
| You → Dev 1 | `IBuffProvider` | You **provide** damage/speed/defense multipliers that Dev 1's combat system queries |
| You → Dev 1 | `IPathProvider` | You **provide** path state (active paths, tiers, abilities) that Dev 1 queries |
| Dev 3 → You | `IRunProgressionEvents` | Dev 3 fires area/boss/island events → you handle rewards, path tier-ups |

## Critical Math Rules

**ALWAYS include numeric examples when creating tasks for these:**

| Rule | Formula | Example |
|------|---------|---------|
| Final Stats | (Base + PathBonus) × Ritual × Trinket × SoulTree | (100 + 20) × 1.3 × 1.1 × 1.05 = 180.18 |
| Ritual Level | L1=1.0×, L2=1.5×, L3=2.0× | L2 burn = 10 × 1.5 = 15 dmg/tick |
| Ritual Stack | 2 same=1.5×, 3 same=2.0× | 2 burn rituals: 15 × 1.5 = 22.5 |
| Ritual Power | **Multiplicative** with itself | Two +10%: 1.1 × 1.1 = **1.21×** NOT 1.2× |
| Path + Ritual | Independent systems, multiplied | Path +30% ATK × Ritual 1.3× = 1.3 × 1.3 = 1.69× |

## Your Task Sequence

```
Phase 1: T006 → T007 → T008, T009 (stats, calculator, paths, currency)
Phase 2: T018 → T019, T020 → T021 (path system, UI, rituals)
Phase 3: T029, T030, T031 (stacking, trinkets, reward UI)
Phase 4: T038 → T039 → T040, T041 (meta, save, hub, inspirations)
Phase 5: T047, T048 → T049 (twins, all families, balance)
Phase 6: T055, T056 (economy, difficulty)
```

## Running a Task

```bash
# Single task through existing pipeline:
/do-task "T007: CharacterStatCalculator — plain C# class. Formula: (Base + Sum(PathBonuses)) × RitualMultiplier × TrinketMultiplier × SoulTreeBonus. Example: (100 HP + 20 path) × 1.3 ritual × 1.1 trinket × 1.05 soul = 180.18"

# Or for a full phase:
/execute-phase 1
```

## Character Stat Reference

| Stat | Brutor | Slasher | Mystica | Viper |
|------|--------|---------|---------|-------|
| HP | **200** | 100 | 50 | 80 |
| DEF | **25** | 8 | 5 | 10 |
| ATK | 0.7 | **2.0** | 0.5 | 0.6m/1.8r |
| SPD | 0.7 | **1.3** | 1.0 | 1.1 |
| MNA | 50 | 60 | **150** | 120 |
| MRG | 2 | 3 | **8** | 6 |
| CRT | 5% | **15%** | 5% | 10% |
| PRS | 1.0 | **1.5** | 0.5 | 0.8 |
