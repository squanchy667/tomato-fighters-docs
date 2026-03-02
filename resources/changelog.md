# Changelog

## [Phase 1] — 2026-03-03 (T008 PathData ScriptableObject — DONE)

### Completed
- **T008: PathData ScriptableObject — 12 Paths** — branch `shared/T008-path-data-so`
  - `PathTierBonuses`: serializable struct with named fields per stat (vs raw float[] — DD-1)
  - `PathData`: SO with 3 incremental tier structs + `GetStatBonusArray(int tier)` for stat calculator
  - Tier bonuses are stored as per-tier deltas; `GetStatBonusArray` accumulates up to the requested tier (DD-3)
  - Tier 3 "Main Path Only" constraint documented via `[Header]` only — no redundant bool flag (DD-2)
  - `GetAbilityIdForTier(int tier)` — clean accessor for ability ID lookup by tier
  - `IPathProvider.MainPath` and `SecondaryPath` updated from `object` placeholders to `PathData`
  - `PathDataCreator`: editor MenuItem (`TomatoFighters/Create All Path Assets`) creates all 12 assets
  - All 12 path assets defined with exact values from CHARACTER-ARCHETYPES.md; run MenuItem in Unity to generate `.asset` files
  - Unblocks: T018 (PathSystem), T028 (Path T1 Ability Execution — Dev 1)

### Design Decisions
- DD-1 (T008): `PathTierBonuses` uses named fields (not `float[]`) — prevents silent index errors, readable in Inspector
- DD-2 (T008): No `isMainExclusive` bool — all T3s are Main-only by design; `[Header]` self-documents, PathSystem enforces
- DD-3 (T008): Incremental tier deltas — each tier stores what it adds; `GetStatBonusArray` sums them — matches design doc wording

---

## [Phase 1] — 2026-03-02 (T007 CharacterStatCalculator + T009 CurrencyManager — DONE)

### Completed
- **T007: CharacterStatCalculator** — branch `pillar2/T007-stat-calculator`
  - Pure C# class (no MonoBehaviour), fully unit-testable without Unity runtime
  - `Calculate(StatModifierInput)` → `FinalStats` — all 10 stats in one call
  - `CalculateSingleStat(StatType, StatModifierInput)` — on-demand single stat
  - Formula: `(base + pathBonus) × ritualMultiplier × trinketMultiplier × soulTreeBonus`
  - Special handling: RangedAttack returns -1 for non-Viper; CritChance clamped to [0,1]
  - `StatModifierInput` flat-array design (indexed by `(int)StatType`) — zero allocations per query
  - `FinalStats` struct with all 10 calculated values
  - Unblocks: T017 (Passives), T025 (HUD), T030 (TrinketSystem), T038 (MetaProgression)

- **T009: CurrencyManager** — branch `pillar2/T009-currency-manager`
  - `CurrencyType` enum added to `Shared/Enums/` (Crystals, ImbuedFruits, PrimordialSeeds)
  - `CurrencyChangeEventData` struct (type, previousAmount, newAmount, delta)
  - `CurrencyManager` MonoBehaviour: injectable via `[SerializeField]`, no singleton
  - Methods: `TryAdd` / `TryRemove` / `GetBalance` / `CanAfford` / `SetBalance` / `ResetPerRunCurrencies`
  - `OnCurrencyChanged` event fired after every successful modification
  - Lock-protected `Dictionary<CurrencyType, int>` backing store (safe for async save/load)
  - Persistence metadata: Crystals persist, ImbuedFruits + PrimordialSeeds reset per run
  - Unblocks: T038 (MetaProgression + SoulTree)

### Design Decisions
- DD-1 (T007): `StatModifierInput` uses flat `float[]` arrays indexed by `(int)StatType` — avoids Dictionary overhead in hot combat path
- DD-1 (T009): `CurrencyType` placed in `Shared/Enums/` (not `Roguelite/`) so World pillar can reference currency types for drop events without crossing pillar boundaries
- DD-2 (T009): MonoBehaviour (not pure C# class) — logic is thin integer arithmetic; no testability benefit from splitting; injection via `[SerializeField]` matches project pattern

---

## [Phase 1] — 2026-03-02 (T002 CharacterController + T003/T004 ComboSystem — DONE)

### Completed
- **T002: CharacterController2D — belt-scroll rework** — 5 code files + 3 editor/test files
  - CharacterMotor: belt-scroll movement (XY ground plane, simulated jump height, no gravity)
  - MovementStateMachine: plain C# state machine (Grounded/Airborne/Dashing)
  - MovementConfig: ScriptableObject with all tuning params
  - CharacterInputHandler: Unity new Input System (InputActionReference)
  - MovementState: 3-state enum
  - 17 edit-mode unit tests for MovementStateMachine

- **T003: InputBufferSystem** — embedded in ComboStateMachine
  - Input buffering during Attacking state, consumed on ComboWindow open

- **T004: ComboSystem — light/heavy chains, finishers** — 7 code files + 1 test file
  - AttackType: Light/Heavy enum
  - ComboState: Idle/Attacking/ComboWindow/Finisher enum
  - ComboStep: serializable struct with branching tree indices (nextOnLight/nextOnHeavy)
  - ComboDefinition: ScriptableObject with flat step array, root indices, per-step window overrides
  - ComboStateMachine: plain C# state machine — input buffering, combo window timer, animation event callbacks
  - ComboController: MonoBehaviour — input → state machine → animation → events
  - ComboDebugUI: debug overlay with auto-advance (simulates animation events when no Animator present)
  - 25 edit-mode unit tests for ComboStateMachine

### Design Decisions
- Belt-scroll movement model (Streets of Rage style): X=horizontal, Y=depth, jump=sprite offset
- No ground collider — grounded is purely `jumpHeight <= 0`
- DD-1: Branching combo tree (L→L→H vs L→L→L) for deep combat variety
- DD-2: Flat array with index pointers — simple inspector authoring
- DD-3: Local C# events (AttackStarted/ComboDropped/FinisherStarted/ComboEnded)
- DD-4: Plain C# state machine with Tick(dt) — testable without Unity runtime

---

## [Phase 1] — 2026-03-02 (updated)

### Completed
- T006: CharacterBaseStats ScriptableObject — branch `pillar2/T006-character-base-stats`
  - `CharacterBaseStats.cs` — 9-field SO (health, defense, attack, rangedAttack, speed,
    mana, manaRegen, critChance, stunRate) with full `[Header]`/`[Tooltip]`/`[Range]` annotations
  - 4 character assets: BrutorStats, SlasherStats, MysticaStats, ViperStats with exact values
    from CHARACTER-ARCHETYPES.md
  - `rangedAttack` is first-class on all 4 characters — throwable system future-proofed (DD-2)
  - `stunRate` / `StatType.StunRate` is canonical name for pressure rate stat (DD-1)
  - `CreateAssetMenu` path: `TomatoFighters/Data/` — sets convention for all future data SOs (DD-3)
  - Fixed GUIDs in `.meta` files so `.asset` files link without requiring Unity open on each machine
  - Unblocks: T007 (CharacterStatCalculator), T008 (PathData)

## [Phase 1] — 2026-03-02

### Completed
- T001: Shared Interfaces, Enums, and Data Structures — 20 files across 4 directories
  - 6 interfaces: ICombatEvents, IBuffProvider, IPathProvider, IDamageable, IAttacker, IRunProgressionEvents
  - 9 enums: CharacterType, DamageType, DamageResponse, PathType, RitualTrigger, RitualFamily, RitualCategory, StatType, TelegraphType
  - 4 data files: DamagePacket, CombatEventData (13 structs), RunEventData (7 structs), PlaceholderTypes
  - 4 assembly definitions enforcing pillar boundaries

### Notes
- Used readonly structs for all event data (immutable by design)
- PlaceholderTypes added for future task dependencies (T005, T008, T021, T028)
- ICombatEvents expanded to 13 events (spec called for 12, added PathAbilityUsed)

---

## v0.0.1 — Project Initialization (2026-03-02)

- Created dual-repo structure (code + docs)
- Defined 4 character archetypes with 12 upgrade paths
- Established interface contracts for 3-pillar architecture
- Created 60-task plan across 6 phases
- Set up AgentPilot config bank with 6 agents and 3 strategies
- Created developer guides for all 3 devs
- Initialized task logbook for shared learning
