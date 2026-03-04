# T008: PathData ScriptableObject — 12 Paths

## Metadata
| Field | Value |
|-------|-------|
| **Phase** | 1 — Foundation |
| **Type** | implementation |
| **Priority** | P0 |
| **Owner** | Dev 2 |
| **Agent** | so-architect |
| **Depends On** | T001, T006 |
| **Blocks** | T018 |
| **Status** | DONE |
| **Completed** | 2026-03-03 |
| **Branch** | `shared/T008-path-data-so` |
| **Merged** | 2026-03-03 → `gal` |
| **Detailed Spec** | [T008-path-data-scriptable-object.md](T008-path-data-scriptable-object.md) |

## Objective
Create the `PathData` ScriptableObject class defining upgrade path definitions with per-tier stat bonuses and ability unlock IDs, and produce all 12 path assets with values from CHARACTER-ARCHETYPES.md. This data drives the Main + Secondary path system that is the core build-crafting mechanic.

## Context
Each character has 3 upgrade paths, each with 3 tiers of progression. During a run, a player selects 1 Main path (tiers 1-3) and 1 Secondary path (tiers 1-2 only). The 3rd path is locked for that run. This creates 6 viable build combinations per character (24 total across all characters).

Path data is purely descriptive — it defines what bonuses and abilities each tier grants. The runtime logic for path selection, tier progression, and ability execution lives in `PathSystem` (T018) and `PathAbilityExecutor` (T028). This SO is the data backbone they both read from.

The `PathType` enum and `StatType` enum should already be defined in T001. The `CharacterBaseStats` SO from T006 provides the base values that path bonuses add to.

## Requirements
1. Create `PathData.cs` ScriptableObject class in `Shared/Data/`
2. Define core fields:
   - `pathName` (string) — display name (e.g., "Warden", "Executioner")
   - `pathType` (PathType enum) — from T001 shared enums
   - `characterType` (CharacterType enum) — which character owns this path
   - `[TextArea] description` (string) — path theme description
3. Define a `TierData` serializable struct with:
   - `statBonuses` — array/dictionary of stat-type-to-value pairs (additive bonuses)
   - `abilityUnlockId` (string) — identifier for the ability unlocked at this tier (e.g., `"Provoke"`, `"IronGuard"`)
   - `abilityName` (string) — display name of the unlocked ability
   - `[TextArea] abilityDescription` (string) — description text for UI
   - `isMainOnly` (bool) — true for T3, indicating only Main path can reach this tier
4. Include an array of 3 `TierData` entries: `tiers[0]` = T1, `tiers[1]` = T2, `tiers[2]` = T3
5. T3 entries must have `isMainOnly = true`
6. Add `[CreateAssetMenu]` attribute for editor asset creation
7. Create all 12 path assets in `ScriptableObjects/Paths/{Character}/` with correct values

### Path Stat Bonuses (from CHARACTER-ARCHETYPES.md)

#### Brutor Paths
| Path | Tier | Bonuses | Ability |
|------|------|---------|---------|
| **Warden** | T1 | +20 HP, +0.2 PRS | Provoke |
| | T2 | +30 HP, +0.3 PRS, +0.1 ATK | AggroAura |
| | T3 | +40 HP, +0.5 PRS, +0.2 ATK | WrathOfTheWarden |
| **Bulwark** | T1 | +30 HP, +5 DEF | IronGuard |
| | T2 | +40 HP, +8 DEF | Retaliation |
| | T3 | +50 HP, +12 DEF, +0.05 CRT | Fortress |
| **Guardian** | T1 | +25 HP, +3 DEF, +20 MNA | ShieldLink |
| | T2 | +35 HP, +5 DEF, +30 MNA | RallyingPresence |
| | T3 | +45 HP, +8 DEF, +40 MNA, +2 MRG | AegisDome |

#### Slasher Paths
| Path | Tier | Bonuses | Ability |
|------|------|---------|---------|
| **Executioner** | T1 | +0.3 ATK, +0.05 CRT | MarkForDeath |
| | T2 | +0.4 ATK, +0.08 CRT | ExecutionThreshold |
| | T3 | +0.5 ATK, +0.12 CRT, +0.3 PRS | Deathblow |
| **Reaper** | T1 | +0.2 ATK, +15 HP | CleavingStrikes |
| | T2 | +0.3 ATK, +20 HP, +0.1 SPD | ChainSlash |
| | T3 | +0.4 ATK, +30 HP, +0.2 SPD | Whirlwind |
| **Shadow** | T1 | +0.2 SPD, +0.05 CRT | PhaseDash |
| | T2 | +0.3 SPD, +0.08 CRT, +10 HP | Afterimage |
| | T3 | +0.4 SPD, +0.12 CRT, +20 HP | ThousandCuts |

#### Mystica Paths
| Path | Tier | Bonuses | Ability |
|------|------|---------|---------|
| **Sage** | T1 | +30 HP, +20 MNA, +2 MRG | MendingAura |
| | T2 | +40 HP, +30 MNA, +3 MRG | PurifyingBurst |
| | T3 | +50 HP, +40 MNA, +4 MRG | Resurrection |
| **Enchanter** | T1 | +20 MNA, +2 MRG, +0.1 SPD | Empower |
| | T2 | +30 MNA, +3 MRG, +0.1 SPD, +15 HP | ElementalInfusion |
| | T3 | +40 MNA, +4 MRG, +0.2 SPD, +25 HP | ArcaneOverdrive |
| **Conjurer** | T1 | +20 HP, +20 MNA, +0.1 ATK | SummonSproutling |
| | T2 | +30 HP, +30 MNA, +0.2 ATK | DeployTotem |
| | T3 | +40 HP, +40 MNA, +0.3 ATK, +2 MRG | SummonGolem |

#### Viper Paths
| Path | Tier | Bonuses | Ability |
|------|------|---------|---------|
| **Marksman** | T1 | +0.3 ranged ATK, +0.05 CRT | PiercingShots |
| | T2 | +0.4 ranged ATK, +0.08 CRT, +10 HP | RapidFire |
| | T3 | +0.5 ranged ATK, +0.12 CRT, +0.1 SPD | Killshot |
| **Trapper** | T1 | +20 HP, +0.1 SPD, +0.2 PRS | HarpoonShot |
| | T2 | +30 HP, +0.2 SPD, +0.3 PRS | TrapNet |
| | T3 | +40 HP, +0.2 SPD, +0.5 PRS, +15 MNA | AnchorChain |
| **Arcanist** | T1 | +30 MNA, +3 MRG | ManaCharge |
| | T2 | +40 MNA, +4 MRG, +0.2 ranged ATK | ManaBlast |
| | T3 | +50 MNA, +5 MRG, +0.3 ranged ATK, +0.05 CRT | ManaOverload |

## File Plan
| File | Description |
|------|-------------|
| `Assets/Scripts/Shared/Data/PathData.cs` | ScriptableObject class with TierData struct, stat bonuses, ability IDs |
| `Assets/ScriptableObjects/Paths/Brutor/WardenPath.asset` | Warden path (aggro tank) |
| `Assets/ScriptableObjects/Paths/Brutor/BulwarkPath.asset` | Bulwark path (counter-tank) |
| `Assets/ScriptableObjects/Paths/Brutor/GuardianPath.asset` | Guardian path (team shield) |
| `Assets/ScriptableObjects/Paths/Slasher/ExecutionerPath.asset` | Executioner path (burst DPS) |
| `Assets/ScriptableObjects/Paths/Slasher/ReaperPath.asset` | Reaper path (AOE cleave) |
| `Assets/ScriptableObjects/Paths/Slasher/ShadowPath.asset` | Shadow path (evasion glass cannon) |
| `Assets/ScriptableObjects/Paths/Mystica/SagePath.asset` | Sage path (healer) |
| `Assets/ScriptableObjects/Paths/Mystica/EnchanterPath.asset` | Enchanter path (buffer) |
| `Assets/ScriptableObjects/Paths/Mystica/ConjurerPath.asset` | Conjurer path (summoner) |
| `Assets/ScriptableObjects/Paths/Viper/MarksmanPath.asset` | Marksman path (sniper) |
| `Assets/ScriptableObjects/Paths/Viper/TrapperPath.asset` | Trapper path (crowd control) |
| `Assets/ScriptableObjects/Paths/Viper/ArcanistPath.asset` | Arcanist path (charge mage) |

## Implementation Notes
- **CRT bonuses stored as decimal** — "+5% CRT" in the design doc becomes `+0.05` in the stat bonus array, consistent with CharacterBaseStats (T006)
- **Viper's ranged ATK bonuses** — Marksman and Arcanist paths grant "ranged ATK" specifically. The stat bonus should target the ranged attack stat, not melee. Consider using a `StatBonusEntry` struct with `StatType stat, float value, bool isRanged` to disambiguate
- **TierData as serializable struct** — Use `[System.Serializable]` so it shows cleanly in the Inspector. Each tier is a self-contained data block
- **Stat bonuses are cumulative per tier** — T2 bonuses are IN ADDITION to T1, not replacements. The calculator (T007) sums all unlocked tier bonuses. Store per-tier incremental bonuses, not cumulative totals
- **isMainOnly on T3** — This is a data flag only. Enforcement happens in PathSystem (T018). T3 `isMainOnly = true` for all 12 paths
- **Ability IDs are strings** — Using strings (not enums) for ability IDs allows the ability execution system (T028) to resolve them dynamically without coupling the data layer to the combat layer
- **Asset creation**: `.asset` files require Unity Editor. The agent creates the C# class; assets can be created via Editor or via an Editor setup script (similar to KingOfOpera's `ProjectSetup.cs` pattern)

## Acceptance Criteria
- [ ] `PathData` ScriptableObject class with `TierData` struct
- [ ] 3 tiers per path with stat bonus arrays
- [ ] Ability unlock ID string per tier
- [ ] `isMainOnly` flag on all T3 entries set to `true`
- [ ] `[CreateAssetMenu]` attribute configured
- [ ] All 12 path assets created with correct stat bonus values
- [ ] Stat bonuses match CHARACTER-ARCHETYPES.md exactly (see tables above)
- [ ] Compiles with zero warnings

## References
- [CHARACTER-ARCHETYPES.md](/design-specs/CHARACTER-ARCHETYPES.md) — Full path descriptions with stat bonuses per tier and ability details
- [TASK_BOARD.md](/TASK_BOARD.md) — T008 entry
- T001 (dependency) — `PathType`, `StatType`, `CharacterType` enums
- T006 (dependency) — `CharacterBaseStats` SO that path bonuses add to
- T018 (blocked) — `PathSystem` reads PathData for selection and tier progression
