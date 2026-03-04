# Changelog

## [Bug Fix] — 2026-03-04 (Damage pipeline not registering hits)

### Problem
Attacks were visually activating (colliders enabling, animations playing) but no damage was being applied to enemies or the player. Three independent bugs combined to make the pipeline appear completely broken.

### Root Causes & Fixes

1. **DebugHealthBar not rendering** (`Scripts/Shared/Components/DebugHealthBar.cs`)
   - **Bug:** Used `Image.Type.Filled` which requires a source sprite to render in Unity 2022. The runtime-created `Texture2D` white pixel, converted to a sprite, wasn't filling correctly.
   - **Fix:** Switched to anchor-based fill using `anchorMax.x = healthRatio` with `Image.Type.Simple`. Health bar now visually updates on every `TakeDamage` call.

2. **PlayerDamageable flash invisible** (`Scripts/Combat/PlayerDamageable.cs`)
   - **Bug:** `GetComponent<SpriteRenderer>()` found the root GameObject's SpriteRenderer, which has no sprite assigned (the visible sprite lives on the `Sprite` child). The damage flash was applying to an invisible renderer.
   - **Fix:** Changed to `GetComponentInChildren<SpriteRenderer>()` to find the visible sprite on the Sprite child GameObject.

3. **Physics2D not detecting re-enabled trigger colliders** (`Scripts/Shared/Components/HitboxDamage.cs`)
   - **Bug:** With `Physics2D.autoSyncTransforms = false` (project default), re-enabling a trigger collider while already overlapping a target did not fire `OnTriggerEnter2D` or `OnTriggerStay2D`. The hitbox GO was being enabled/disabled each attack, but the physics system never registered the overlap.
   - **Fix:** Added `Physics2D.SyncTransforms()` and `Rigidbody2D.WakeUp()` calls in `HitboxDamage.OnEnable()` to force the physics engine to recognize the newly-enabled collider.

### Additional Changes
- **TestDummyEnemy.cs** (`Scripts/World/`): Added `OnDamaged()` override with white sprite flash for visual hit-confirm feedback
- **HitboxManager.cs** (`Scripts/Combat/Hitbox/`): Added 3 diagnostic `Debug.Log` lines for pipeline tracing (temporary, remove before ship)
- **HitboxDamage.cs** (`Scripts/Shared/Components/`): Added diagnostic log in `OnTriggerStay2D` (temporary)
- **DamagePipelineDiagnostic.cs** (`Assets/` root): New temporary debug tool that validates the entire damage pipeline on scene `Start()` — checks layers, collider states, component wiring, Physics2D settings. Lives outside asmdef folders intentionally.
- **MovementTestSceneCreator.cs** (`Editor/Prefabs/`): Now auto-adds `DamagePipelineDiagnostic` to Main Camera via string-based `Type.GetType()` lookup

### Files Modified
- `unity/TomatoFighters/Assets/Scripts/Shared/Components/DebugHealthBar.cs`
- `unity/TomatoFighters/Assets/Scripts/Shared/Components/HitboxDamage.cs`
- `unity/TomatoFighters/Assets/Scripts/Combat/PlayerDamageable.cs`
- `unity/TomatoFighters/Assets/Scripts/Combat/Hitbox/HitboxManager.cs`
- `unity/TomatoFighters/Assets/Scripts/World/TestDummyEnemy.cs`
- `unity/TomatoFighters/Assets/Editor/Prefabs/MovementTestSceneCreator.cs`
- `unity/TomatoFighters/Assets/DamagePipelineDiagnostic.cs` (NEW)

### Lessons Learned
- Unity 2022's `Image.Type.Filled` requires a properly-configured source sprite; runtime-created textures don't work reliably with fill mode
- Always use `GetComponentInChildren<T>()` when the visual renderer might be on a child (common in prefabs with root+sprite+shadow structure)
- When `autoSyncTransforms` is off, toggling collider GameObjects requires manual `Physics2D.SyncTransforms()` — this is a known Unity gotcha for enable/disable hitbox patterns
- Diagnostic tools (`DamagePipelineDiagnostic`) that validate the full pipeline on Start are invaluable for catching integration issues early

---

## [Phase 2] — 2026-03-03 (T020 RitualData — DONE)

### Completed
- **T020: RitualData ScriptableObject + Families** — branch `pillar2/T020-ritual-data-so`
  - `RitualData` SO in `Roguelite/` — placement overrides task board's `Shared/Data/` (DD-3: no Shared interface exposes RitualData directly)
  - `RitualLevelData` struct: `baseValue`, `maxStacks`, `stackingMultiplier`, `ritualPower` — three explicit level fields mirroring PathTierBonuses (DD-2)
  - `string effectId` dispatch key — matches PathData ability ID pattern; RitualSystem (T021) wires a `Dictionary<string, Action>` (DD-1)
  - `BelongsToFamily(RitualFamily)` — handles Twin ritual dual-family membership
  - `GetLevelData(int)` — clean level accessor, clamped 1–3
  - `RitualDataCreator` editor MenuItem (`TomatoFighters/Create Ritual Assets`) — generates 8 assets
  - Fire family: Burn (Core/OnStrike), Blazing Dash (General/OnDash), Flame Strike (Enhancement/OnFinisher), Ember Shield (Enhancement/OnDeflect)
  - Lightning family: Chain Lightning (Core/OnStrike), Lightning Strike (General/OnSkill), Shock Wave (Enhancement/OnFinisher), Static Field (Enhancement/OnTakeDamage)
  - Unblocks: T021 (RitualSystem)

### Design Decisions
- DD-1 (T020): `string effectId` — 36+ rituals would bloat an enum; string keys match PathData's ability ID pattern; dispatch via Dictionary in RitualSystem
- DD-2 (T020): Three explicit `RitualLevelData` structs — mirrors PathTierBonuses; level multipliers (1.0/1.5/2.0) are fixed constants in RitualStackCalculator (T029)
- DD-3 (T020): `Roguelite/` not `Shared/Data/` — `IBuffProvider.GetTriggerEffects` returns `OnTriggerEffect`, never `RitualData`; Roguelite is the rightful owner

### Tests Added
- `CharacterStatCalculatorTests.cs` — 17 edit-mode unit tests for T007 formula (path, ritual, trinket, soul tree stacking; crit clamp; Viper rangedAttack)
- `PathSystemTests.cs` — 24 edit-mode unit tests for T018 selection rules, tier progression, run lifecycle, IPathProvider, and events

---

## [Phase 2] — 2026-03-03 (T018 PathSystem — DONE)

### Completed
- **T018: PathSystem — Selection + Tier Progression** — branch `pillar2/T018-path-system`
  - `PathSystem` MonoBehaviour implementing `IPathProvider` — single source of truth for run path state
  - `SelectMainPath` / `SelectSecondaryPath` return bool, no throws — UI responsible for valid options (DD-3)
  - 5-condition guard on `SelectSecondaryPath`: null, wrong character, no main yet, already selected, same as main
  - `HandleBossDefeated` → both paths T1→T2; `HandleIslandCompleted` → main only T2→T3
  - `TryAdvanceTier` private helper: idempotent — calling twice at same tier is a no-op
  - `IRunProgressionEvents` subscribed via `[SerializeField] MonoBehaviour` cast in `Awake()`, null-safe (DD-2)
  - Own `Action<PathSelectedData>` + `Action<PathTierUpData>` events — not piped through `IRunProgressionEvents` (DD-5)
  - `ResetForNewRun()` clears all four state fields
  - `TomatoFighters.Paths.asmdef` added — Paths folder was missing assembly definition
  - Unblocks: T019 (PathSelectionUI), T028 (Path T1 Ability Execution), T041 (InspirationSystem)

### Design Decisions
- DD-1 (T018): MonoBehaviour (not pure C# class) — holds per-run state, Unity lifecycle for subscribe/unsubscribe, `[SerializeField]` injection pattern matches CurrencyManager
- DD-2 (T018): Subscribe to `IRunProgressionEvents` via SerializeField cast — null-safe until Dev 3 ships RunManager; direct `HandleBossDefeated/HandleIslandCompleted` methods remain callable for interim testing
- DD-3 (T018): Selection returns bool, no throws — combat/game code must never throw
- DD-4 (T018): PathSystem does not hold available path list — "which options to show" is a UI concern (T019)
- DD-5 (T018): Own C# events — C# events can only be raised by their declaring class; uses same shared data types

---

## [Tooling] — 2026-03-03 (Plan → Execute workflow)

### Changed
- **`/plan-task`** — Step 4 now **mandates** writing the spec to disk before finishing. Step 5 (offering to execute in the same session) removed. Planning agent always writes `## Design Decisions` to the spec file and directs you to execute in a new session.

### Added
- **`/save-spec`** — New command to invoke mid-conversation during `/plan-task`. Triggers spec-writing and stops. Use when the planning agent jumps ahead to "should I execute?" before persisting decisions.

### Workflow
```
Session 1:  /plan-task TXXX  →  discuss  →  /save-spec  →  done
Session 2:  /task-execute TXXX  →  clean context execution
```

---

## [Misc] — 2026-03-03 (Animation updates + editor fixes)

### Changes
- Animation updates: purple mage assets integrated
- `AttackData.hitboxId` serialization fix
- Project settings updates
- `DebugHealthBar` temp component added to `Shared/Components/` for visual debug (replaced by T025)
- `PlayerPrefabCreator` updated: standing collider (0.8×1.2), PlayerHurtbox layer, PlayerDamageable + DebugHealthBar baked in, missing script cleanup + hitbox child repair
- `TestDummyPrefabCreator` updated: DebugHealthBar (red) baked into prefab
- `MovementTestSceneCreator` updated: DebugHealthBar on both player (green) and enemy (red)

---

## [Phase 1] — 2026-03-03 (T011 EnemyBase — DONE)

### Completed
- **T011: EnemyBase — IDamageable + IAttacker** — branch `pillar3/T011-enemy-base`
  - `Shared/Components/HitboxDamage.cs`: Moved from `Combat/Hitbox/` to `Shared/Components/` — shared trigger detection for both player and enemy hitboxes
  - `World/EnemyData.cs`: ScriptableObject with `[CreateAssetMenu]` — maxHealth, pressureThreshold, stunDuration, invulnerabilityDuration, knockbackResistance, movementSpeed, AttackData[] attacks
  - `World/EnemyBase.cs`: Abstract MonoBehaviour implementing IDamageable (full) + IAttacker (virtual stubs). Pressure meter, knockback via Rigidbody2D.AddForce, stun/recovery coroutines, invulnerability blink, death event, virtual hooks (OnDamaged, OnStunned, OnDeath, OnRecovery)
  - `World/TestDummyEnemy.cs`: Concrete test enemy — timer-based attack loop (3s interval, 0.4s telegraph, 0.6s active), coroutine-driven hitbox activation, IAttacker overrides during attack window
  - `Combat/PlayerDamageable.cs`: Stub IDamageable on player — 100 HP, logs damage, red sprite flash, knockback via Rigidbody2D
  - `World/DebugHealthBar.cs`: Temporary floating HP bar using SpriteRenderer fill (replaced by T025 HUD)
  - `Editor/Prefabs/TestDummyPrefabCreator.cs`: Creates TestDummy.prefab, EnemyData SO, DummyPunch AttackData SO, WhiteSquare debug sprite
  - `Editor/Prefabs/MovementTestSceneCreator.cs`: Updated to spawn TestDummy at (3,0) and add PlayerDamageable to player
  - `Editor/SetupMysticaCharacter.cs`: Updated HitboxDamage import + added red debug visuals on player hitboxes
  - `Editor/TomatoFighters.Editor.asmdef`: Added TomatoFighters.World reference

### Design Decisions
- DD-1 (T011): EnemyData lives in `World/EnemyData.cs` — Dev 3's pillar domain
- DD-2 (T011): Attack timer in TestDummyEnemy subclass, not EnemyBase — base stays abstract and clean
- DD-3 (T011): HitboxDamage moved to `Shared/Components/` — zero Combat dependencies, shared by both pillars
- DD-4 (T011): Stub PlayerDamageable in Combat pillar — minimal stub for bidirectional damage testing
- DD-5 (T011): WhiteSquare PNG written to disk (not runtime Texture2D) — survives prefab serialization

### Notes
- Task counter: 12/60 (Phase 1: 10/13, Phase 2: 2/12)
- T011 unblocks: T013 (TestScene), T022 (BasicEnemyAI)
- Bidirectional damage pipeline verified: player → enemy (HitboxManager) and enemy → player (TestDummyEnemy + HitboxDamage + PlayerDamageable)
- Debug visuals: orange body sprite for enemy, semi-transparent red overlay for active hitboxes (both player and enemy)

---

## [Phase 2] — 2026-03-03 (T015 HitboxManager — DONE)

### Completed
- **T015: HitboxManager** — branch `pillar1/T015-hitbox-manager`
  - `Combat/Hitbox/HitDetectionData.cs`: Readonly struct payload (IDamageable target, AttackData, Vector2 hitPoint, CharacterType attacker)
  - `Combat/Hitbox/HitboxDamage.cs`: MonoBehaviour on each hitbox child GO. `HashSet<IDamageable>` per-activation tracking (cleared on OnEnable). `OnTriggerEnter2D` detection, fires event with target + hitPoint. Pure detection — no damage resolution
  - `Combat/Hitbox/HitboxManager.cs`: Orchestrator on player root. `ActivateHitbox()`/`DeactivateHitbox()` Animation Event callbacks. Reads `hitboxId` from current ComboStep's AttackData, looks up `Hitbox_{id}` child GO. Subscribes to HitboxDamage events, forwards hit-confirm to `ComboController.OnHitConfirmed()`. Temporary damage shim (builds DamagePacket, calls IDamageable.TakeDamage) clearly marked for T016 replacement
  - `Shared/Data/AttackData.cs`: Added `string hitboxId` field in Animation & Timing section
  - `Editor/SetupMysticaCharacter.cs`: One-click prefab setup — creates 3 hitbox children (Burst, BigBurst, Bolt), wires HitboxManager + ComboController, assigns hitboxId on all 5 Mystica AttackData assets
  - `Editor/AssignAllHitboxIds.cs`: Bulk assigns hitboxId on all 26 AttackData assets across 4 characters
  - `developer/unity-editor-scripts.md`: Documentation for all editor setup scripts

### Design Decisions
- DD-2 (T015): AttackData holds `hitboxId`, Animation Events call generic `ActivateHitbox()` — no string args. HitboxManager reads hitboxId from current combo step's AttackData
- DD-3 (T015): HashSet tracking by `IDamageable` (not GameObject) — prevents double-damage from entities with multiple colliders
- DD-4 (T015): Event chain: HitboxDamage.OnHitDetected → HitboxManager → ComboController.OnHitConfirmed()
- DD-5 (T015): Detection-only pattern — HitboxDamage reports collisions, temp shim in HitboxManager applies damage directly until T016

### Notes
- Task counter: 11/60 (Phase 1: 9/13, Phase 2: 2/12)
- T015 unblocks: T016 (DefenseSystem), T017 (Character Passives)
- T018 (PathSystem) and T020 (RitualData) appear completed on `gal` branch per git log — board may need separate update

---

## [Phase 2] — 2026-03-03 (T014 ComboSystem — All 4 Characters — DONE)

### Completed
- **T014: ComboSystem — All 4 Characters** — branch `tal`
  - `Editor/CreateAllCharacterAttacks.cs`: Unified editor script creating 22 AttackData assets for Brutor (7), Slasher (8), Viper (6), Mystica extras (1)
  - `Editor/CreateComboDefinitions.cs`: Creates 3 ComboDefinition SOs (Slasher 8-step, Mystica 5-step, Viper 6-step) + 4 ComboInteractionConfig SOs
  - Brutor's existing 7-step tree from T004 retained; AttackData assets now created for all his steps
  - No changes to combo infrastructure code (ComboStateMachine, ComboController, ComboStep, ComboDefinition)

### Design Decisions
- DD-1 (T014): Unified editor script (not per-character) — single menu item creates all assets
- DD-2 (T014): Slasher 8-step (deepest, Heavy→Light re-entry unique to him), Mystica 5-step (shallowest, forgiving windows), Viper 6-step (medium, ranged flavor)
- DD-3 (T014): Tuning values encode character identity — Brutor slow/heavy (20-30f), Slasher fast/snappy (12-18f), frame values are design intent for T024 animation work
- DD-4 (T014): Per-character cancel configs — Slasher's dash doesn't reset combo (mid-chain dash), Mystica prioritizes jump-cancel (blink escape), Viper can move during normal attacks (kiting)

### Notes
- Task counter: 10/60 (Phase 1: 9/13, Phase 2: 1/12)
- T014 unblocks: T015 (HitboxManager), T016 (DefenseSystem), T017 (Character Passives)
- Phase 2 is now IN_PROGRESS

---

## [Phase 1] — 2026-03-03 (T005 AttackData ScriptableObject — DONE)

### Completed
- **T005: AttackData ScriptableObject** — branch `tal` (pillar1/T005-attack-data-so)
  - `Shared/Data/AttackData.cs`: Universal attack data container with 5 inspector groups (Identity, Damage, Animation & Timing, Telegraph & State, Effects). 15+ fields with `[Header]`, `[Tooltip]`, `[Range]` attributes.
  - Mystica 4-attack set via `Editor/CreateMysticaAttacks.cs` (menu: `Tools > TomatoFighters > Create Mystica Attacks`):
    - MysticaStrike1 (Magic Burst 1): 0.6× dmg, 16 frames, kb (1.5, 0)
    - MysticaStrike2 (Magic Burst 2): 0.8× dmg, 18 frames, kb (2.0, 0.3)
    - MysticaStrike3 (Magic Burst 3): 1.0× dmg, 22 frames, kb (3.0, 0.5)
    - MysticaArcaneBolt (Arcane Bolt): 1.4× dmg, 30 frames, kb (2.0, 1.0)
  - `ComboStep.cs`: Added `AttackData attackData` field (DD-1: keeps existing `damageMultiplier` as combo-specific override)
  - `IAttacker.cs`: `CurrentAttack` changed from `object` to `AttackData` (DD-3: resolves T001 TODO)

### Design Decisions
- DD-1 (T005): ComboStep dual scaling — `attackData.damageMultiplier × comboStep.damageMultiplier` — same AttackData reusable across different combo trees
- DD-2 (T005): Mystica first (not Brutor) — her fast animations (16-22f) and low ATK (0.5) test the damage pipeline at the low end
- DD-3 (T005): IAttacker.CurrentAttack typed to AttackData — no existing implementors, safe non-breaking change

### Notes
- Task counter: 10/60 (Phase 1: 10/13)
- T005 unblocks: T014 (ComboSystem all characters), T023 (Enemy Attack Patterns)
- Remaining Phase 1: T010 (WaveManager), T011 (EnemyBase), T012 (CameraController)

---

## [Phase 1] — 2026-03-03 (T004 Basic Strike Combo Chain — DONE)

### Completed
- **T004: IN PROGRESS → DONE** — All acceptance criteria resolved. The InputBufferSystem integration checkbox was a phantom blocker: T003's buffer IS the 1-slot buffer inside ComboStateMachine (BufferedInput property, lines 22/95/117). Cancel inputs use instant checks (DD-10). Animation-driven timing callbacks are wired and ready for T005's AttackData SO. Task counter: 9/60.

### Closure Notes
- Remaining deferred items (AttackData reference on ComboStep, frame-data-driven timing) are T005's responsibility, not T004's. T004 has the hooks ready — T005 fills in the data.
- The lesson: when T003 changed scope from standalone to embedded, T004's acceptance criteria should have been updated immediately. This caused T004 to appear incomplete for a day.

---

## [Phase 1] — 2026-03-03 (Docs sync)

### Status Updates
- **T006: PENDING → DONE** — code and 4 character assets existed since 2026-03-02 but board/spec were never updated
- **T007: PENDING → DONE** — CharacterStatCalculator, FinalStats, StatModifierInput all exist; board/spec were never updated
- **T004: IN PROGRESS** — hit-confirm, cancel flags (dash/jump), movement lock, stagger/death resets now implemented. Remaining: InputBufferSystem integration (deferred). Cancel input buffering added to ComboController with `TryDashCancel`/`TryJumpCancel` and `ComboInteractionConfig` SO for priority/reset config.

### Deviations Flagged (require human review)
- T002 spec lists files under `Characters/` but actual code is under `Combat/Movement/`
- T001 has undocumented additions: `AttackMode.cs` enum, `TomatoFighters.Characters.asmdef`
- T003 acceptance criteria were retroactively simplified (changelog 2026-03-02 already flagged this)

---

## [Phase 1] — 2026-03-02 (Docs sync + T004 audit)

### Status Updates
- **T004: reopened as IN PROGRESS** — audit revealed acceptance criteria were retroactively rewritten to hide missing features. Original spec requires: hit-confirm system, cancel flags, movement lock, InputBufferSystem integration, stagger/death resets. None were implemented. Acceptance criteria restored to originals with honest status.

### Design Decisions (T004 planning session)
- DD-5: State machine exposes cancel properties, Controller orchestrates
- DD-6: ComboController calls motor.SetAttackLock(bool), motor is passive
- DD-7: ComboInteractionConfig SO — broad config for cancels, resets, movement lock
- DD-8: No animator = Debug.LogError on Awake, not silent fail
- DD-9: ComboDebugUI stays dumb — no config awareness
- DD-10: InputBufferSystem deferred to T003 reopen — cancel inputs are instant checks

### Notes
- T003 also likely needs reopening — original spec called for standalone InputBufferSystem with circular buffer, BufferedInputType enum (Strike/Skill/Arcana/Dash/Jump), TryConsumeInput API. Was retroactively marked DONE with rewritten acceptance criteria matching the 1-slot buffer inside ComboStateMachine.

---

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
