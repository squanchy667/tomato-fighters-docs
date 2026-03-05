# Tomato Fighters — Task Board

> 20/60 tasks DONE | 6 phases | 3 developers + AgentPilot

---

## Phase 1: Foundation
> Status: IN_PROGRESS | Tasks: 12/13 | Weeks: 1-2
> Goal: Shared contracts established, one character moves and hits a dummy enemy, camera follows

### T001: Shared Interfaces, Enums, and Data Structures [DONE]
- **Type:** implementation | **Priority:** P0 | **Owner:** ALL | **Depends on:** none
- **Files:** `Shared/Interfaces/ICombatEvents.cs`, `Shared/Interfaces/IBuffProvider.cs`, `Shared/Interfaces/IPathProvider.cs`, `Shared/Interfaces/IDamageable.cs`, `Shared/Interfaces/IAttacker.cs`, `Shared/Interfaces/IRunProgressionEvents.cs`, `Shared/Enums/*.cs`, `Shared/Data/*.cs`
- **Description:** Define ALL cross-pillar interface contracts, shared enums (CharacterType, PathType, StatType, DamageType, RitualTrigger, TelegraphType, RitualFamily, RitualCategory), and data structures (DamagePacket, AttackData base, StrikeEventData, etc.). This is a sync session — all 3 devs together.
- **Acceptance:**
  - [x] ICombatEvents with all 12 event signatures
  - [x] IBuffProvider with path-extended queries
  - [x] IPathProvider with tier + ability queries
  - [x] IDamageable + IAttacker interfaces
  - [x] IRunProgressionEvents with path selection events
  - [x] All shared enums defined
  - [x] DamagePacket struct defined
  - [x] Compiles with zero warnings

### T002: CharacterController2D — Brutor First [DONE]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T001
- **Files:** `Characters/CharacterMotor.cs`, `Characters/MovementStateMachine.cs`, `Characters/MovementConfig.cs`, `Characters/CharacterInputHandler.cs`, `Characters/MovementState.cs`
- **Description:** Belt-scroll character controller (Streets of Rage style). XY ground plane, simulated jump height via sprite offset, dash with i-frames. Rigidbody2D with gravityScale=0. All values configurable via MovementConfig ScriptableObject. 17 unit tests.
- **Acceptance:**
  - [x] 8-directional movement with configurable speed
  - [x] Jump with simulated gravity (belt-scroll model)
  - [x] Dash with i-frame support (delegate to DefenseSystem later)
  - [x] Brutor's armored dash (super-armor flag)
  - [x] Rigidbody2D physics, never transform.position
  - [x] All values in Inspector via SerializeField

### T003: InputBufferSystem [DONE]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T001
- **Files:** Input buffering embedded in `Combat/Combo/ComboStateMachine.cs`
- **Description:** Input buffering implemented directly inside ComboStateMachine rather than as a standalone system. Buffers one input during Attacking state, consumes on ComboWindow open. Clears on combo drop/reset.
- **Acceptance:**
  - [x] Input buffering during attack animations
  - [x] Pre-buffering during active animations
  - [x] Buffer clears on state reset
  - [x] Compiles with zero warnings

### T004: Basic Strike Combo Chain [DONE]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T002, T003
- **Files:** `Combat/Combo/AttackType.cs`, `Combat/Combo/ComboState.cs`, `Combat/Combo/ComboStep.cs`, `Combat/Combo/ComboDefinition.cs`, `Combat/Combo/ComboStateMachine.cs`, `Combat/Combo/ComboController.cs`, `Combat/Combo/ComboDebugUI.cs`
- **Description:** Branching combo tree with light/heavy paths. Brutor: 7-step tree (L→L→L sweep, L→H launcher, H→H ground pound). Plain C# ComboStateMachine with Tick(dt) for testability. Animation-event-driven transitions. Hit-confirm cancel system.
- **Acceptance:**
  - [x] Branching combo tree with light/heavy paths per step
  - [x] 7-step branching tree for Brutor with finishers
  - [x] Input window per step (configurable)
  - [x] ComboDefinition ScriptableObject with flat step array
  - [x] Plain C# ComboStateMachine testable without Unity runtime
  - [x] Hit-confirm callback (`OnHitConfirmed`) with dash-cancel and jump-cancel
  - [x] `canDashCancelOnHit` / `canJumpCancelOnHit` fields on ComboStep
  - [x] InputBufferSystem integration (T003 buffer is the 1-slot buffer inside ComboStateMachine — same system, see DD-10)
  - [x] Movement lock integration (CharacterMotor subscribes to combo events)
  - [x] Combo resets on stagger/death (not just window expiry)

### T005: AttackData ScriptableObject [DONE]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T001
- **Files:** `Shared/Data/AttackData.cs`, `ScriptableObjects/Attacks/Mystica/*.asset`, `Editor/CreateMysticaAttacks.cs`
- **Description:** ScriptableObject definition for attack data: damage multiplier, knockback force, launch force, animation clip ref, hitbox timing (start frame, active frames), telegraph type (Normal/Unstoppable), VFX/SFX refs. Create Mystica's 4-attack set (DD-2: Mystica first). ComboStep gains AttackData reference (DD-1). IAttacker.CurrentAttack typed to AttackData (DD-3).
- **Acceptance:**
  - [x] AttackData SO with all required fields
  - [x] Mystica attack set: 4 assets (Strike1, Strike2, Strike3, ArcaneBolt)
  - [x] TelegraphType field (Normal/Unstoppable)
  - [x] CreateAssetMenu attribute for easy creation
  - [x] ComboStep updated with AttackData field (DD-1)
  - [x] IAttacker.CurrentAttack updated from object to AttackData (DD-3)
  - [x] Editor script for programmatic asset creation

### T006: CharacterBaseStats ScriptableObject [DONE]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 2 | **Depends on:** T001
- **Files:** `Shared/Data/CharacterBaseStats.cs`, `ScriptableObjects/Characters/BrutorStats.asset`, `SlasherStats.asset`, `MysticaStats.asset`, `ViperStats.asset`
- **Description:** ScriptableObject defining the 8 base stats per character (HP, DEF, ATK, SPD, MNA, MRG, CRT, PRS). Create all 4 character stat assets with the values from CHARACTER-ARCHETYPES.md. Include passive ability identifier.
- **Acceptance:**
  - [x] CharacterBaseStats SO with 8 stat fields + passive ID
  - [x] 4 character assets with correct stat values
  - [x] Stat values match CHARACTER-ARCHETYPES.md exactly

### T007: CharacterStatCalculator [DONE]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T006
- **Files:** `Paths/CharacterStatCalculator.cs`
- **Description:** Plain C# class (not MonoBehaviour) for testability. Calculates final stats: Base + Sum(PathBonuses) then multiply by RitualMultiplier, TrinketMultiplier, SoulTreeBonus. Each stat independently calculated. Returns a FinalStats struct.
- **Acceptance:**
  - [x] Pure C# class, not MonoBehaviour
  - [x] Correct formula: (Base + PathBonuses) * Ritual * Trinket * SoulTree
  - [x] Independent per-stat calculation
  - [x] Returns FinalStats with all 8 stats
  - [x] Unit testable with mock inputs

### T008: PathData ScriptableObject — 12 Paths [DONE]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 2 | **Depends on:** T001, T006
- **Files:** `Shared/Data/PathData.cs`, `Shared/Data/PathTierBonuses.cs`, `Editor/PathDataCreator.cs`, `ScriptableObjects/Paths/{Character}/{PathName}Path.asset` (12 total)
- **Description:** ScriptableObject for upgrade path definitions. Fields: PathType enum, stat bonuses per tier (T1/T2/T3), ability unlock IDs per tier, description text. Create all 12 path assets matching CHARACTER-ARCHETYPES.md.
- **Acceptance:**
  - [x] PathData SO with tier definitions (3 tiers)
  - [x] Stat bonus arrays per tier (named PathTierBonuses struct — DD-1)
  - [x] Ability unlock ID per tier
  - [x] All 12 path assets defined in PathDataCreator editor script with correct values
  - [x] T3 marked as Main-only via [Header] — no runtime bool (DD-2)
  - [x] IPathProvider updated: object placeholders replaced with PathData

### T009: CurrencyManager [DONE]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T001
- **Files:** `Roguelite/CurrencyManager.cs`
- **Description:** Central authority for 3 currencies: Crystals (common), Imbued Fruits (rare), Primordial Seeds (rare). Fires events on change. Thread-safe modifications. Crystals persist between runs, fruits/seeds earned per run.
- **Acceptance:**
  - [x] 3 currency types managed
  - [x] Events fired on any currency change
  - [x] Add/Remove/Check methods
  - [x] Persistence flag per currency type

### T010: WaveManager [DONE]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 3 | **Depends on:** T001
- **Files:** `World/WaveManager.cs`, `World/LevelBound.cs`, `Shared/Data/WaveData.cs`, `Shared/Data/EnemySpawnData.cs`, `Shared/Events/VoidEventChannel.cs`, `Shared/Events/IntEventChannel.cs`, `Editor/WaveManagerAssetsCreator.cs`
- **Branch:** `ofek` | **Completed:** 2026-03-05
- **Description:** Spawn enemies in configurable waves. Camera stops at LevelBound until wave cleared. Configurable enemy composition per wave (EnemySpawnData list). Fires events on wave start, wave clear, area complete via SO event channels. Area complete triggers reward selection.
- **Acceptance:**
  - [x] Wave list with configurable enemy spawns
  - [x] LevelBound camera stops
  - [x] Events: OnWaveStart, OnWaveCleared, OnAreaComplete
  - [x] Optional waves (side paths)

### T011: EnemyBase — IDamageable + IAttacker [DONE]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T001
- **Files:** `World/EnemyBase.cs`, `World/EnemyData.cs`, `World/TestDummyEnemy.cs`, `Combat/PlayerDamageable.cs`, `Shared/Components/HitboxDamage.cs`
- **Description:** Abstract base class all enemies inherit. Implements IDamageable (health, pressure meter, knockback, launch, invulnerability blink) and IAttacker (virtual stubs). EnemyData SO for stats. TestDummyEnemy for combat testing. PlayerDamageable stub for bidirectional damage. HitboxDamage moved to Shared.
- **Acceptance:**
  - [x] Implements IDamageable fully
  - [x] Implements IAttacker fully
  - [x] Health + pressure meter
  - [x] Knockback via Rigidbody2D
  - [x] Invulnerability after stun recovery (blink white)
  - [x] Reads stats from EnemyData SO

### T012: CameraController2D [DONE]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 3 | **Depends on:** T001
- **Files:** `World/CameraController2D.cs`
- **Branch:** `ofek` | **Completed:** 2026-03-05
- **Description:** Side-scroll follow with configurable leading. Level bound stops (integrates with WaveManager via SO event channels). Smooth damping. Zoom-in on stun events (listens to VoidEventChannel, doesn't reference PressureSystem). Co-op framing for 2 players. Dynamic bounds via SetBounds(). Gizmo visualization.
- **Acceptance:**
  - [x] Smooth follow with configurable leading
  - [x] Respects level bounds
  - [x] Zoom on stun event (via SO event channel)
  - [x] All values configurable in Inspector

### T013: Basic Test Scene [IN_PROGRESS]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 3 | **Depends on:** T002, T010, T011, T012
- **Files:** `Scenes/TestArena.unity`
- **Description:** Simple test scene: flat ground, walls on both sides, camera setup, 1 player spawn, 2-3 dummy enemies. Used for Phase 1 integration testing.
- **Acceptance:**
  - [ ] Flat ground with collision
  - [ ] Walls on both sides
  - [ ] Player spawn point
  - [ ] 2-3 dummy enemies (EnemyBase with basic stats)
  - [ ] Camera following player

---

## Phase 2: Core Combat + Path Framework
> Status: IN_PROGRESS | Tasks: 7/12 | Weeks: 3-4
> Goal: All 4 characters playable with basic combos, path selection works, fight one wave

### T014: ComboSystem — All 4 Characters [DONE]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T004, T005
- **Files:** `Combat/ComboSystem.cs` (extended), character-specific AttackData assets
- **Description:** Extend ComboSystem for all 4 characters. Brutor: 3-hit shield bash. Slasher: 4-hit fast slash. Mystica: 3-hit projectile burst. Viper: 3-shot burst. Each with unique finishers, timing, and cancel routes. Combo tree branching per character.
- **Acceptance:**
  - [x] 4 distinct combo trees
  - [x] Character-specific timing windows
  - [x] All AttackData assets created
  - [x] Skill/Heavy attacks per character

### T015: HitboxManager [DONE]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T014
- **Files:** `Combat/HitboxManager.cs`
- **Description:** Activate/deactivate attack colliders via Animation Events. Never in Update(). Reports hit-confirm to ComboSystem for cancel enabling. Supports multiple hitbox shapes per attack. Collision layer filtering. **Unlocks cancel system end-to-end testing** — wires collision → `ComboController.OnHitConfirmed()` → cancel window opens.
- **Acceptance:**
  - [x] Animation Event-driven activation
  - [x] Hit-confirm callback to ComboSystem
  - [x] Multiple hitbox shapes supported
  - [x] Layer filtering (player vs enemy)
  - [x] Cancel system integration tests: hit → cancel window → dash/jump cancel executes (see `.claude/dumps/cancel-buffer-tests-dump.md`)

### T016: DefenseSystem — Deflect/Clash/Dodge [DONE]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T014
- **Files:** `Combat/Defense/DefenseSystem.cs`, `Combat/Defense/DefenseResolver.cs`, `Combat/Defense/DefenseConfig.cs`, `Combat/Defense/DefenseBonus.cs`, `Combat/Defense/Bonuses/*`, `Shared/Interfaces/IDefenseProvider.cs`
- **Branch:** `pillar1/T016-defense-system` | **Merged:** 2026-03-03 → `tal`
- **Description:** Per-character defense timing. Deflect (on dash, generous: 0-150ms), Clash (on heavy, tight: 20-80ms), Dodge (i-frames: 50-300ms). Returns DamageResponse enum. Character-specific bonuses: Brutor no-slideback on deflect, Slasher crit-on-deflect, Mystica mana-on-deflect, Viper projectile-reflect.
- **Acceptance:**
  - [x] Deflect with configurable window per character
  - [x] Clash with tighter window
  - [x] Dodge i-frames
  - [x] Unstoppable attacks bypass deflect/clash
  - [x] Character-specific deflect bonuses
  - [x] DamageResponse enum (Hit, Deflected, Clashed, Dodged)

### T017: Character Passives [DONE]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 1 | **Depends on:** T007, T014
- **Branch:** `combat/T017-character-passives` | **Completed:** 2026-03-04
- **Files:** `Characters/PassiveAbilitySystem.cs`, `Characters/Passives/ThickSkin.cs`, `Bloodlust.cs`, `ArcaneResonance.cs`, `DistanceBonus.cs`, `PassiveConfig.cs`, `IPassiveAbility.cs`, `Shared/Data/HitContext.cs`, `Shared/Interfaces/IPassiveProvider.cs`
- **Description:** Implement 4 character passive abilities. Brutor: Thick Skin (15% DR, 40% less knockback). Slasher: Bloodlust (3% ATK per hit, 10 stacks, 3s decay). Mystica: Arcane Resonance (+5% damage per cast to allies, 3 stacks). Viper: Distance Bonus (+2%/unit, max +30%).
- **Acceptance:**
  - [x] 4 passive abilities implemented
  - [x] Stacking/decay where applicable
  - [x] Reads from CharacterStatCalculator
  - [x] Fires through IBuffProvider pipeline

### T018: PathSystem — Selection + Tier Progression [DONE]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 2 | **Depends on:** T008
- **Files:** `Paths/PathSystem.cs`, `Paths/TomatoFighters.Paths.asmdef`
- **Branch:** `pillar2/T018-path-system` | **Merged:** 2026-03-03 → `gal`
- **Description:** Manages path selection state per character per run. Enforces: 1 Main + 1 Secondary, cannot select same path twice, 3rd path locked. Tier progression: T1 on select, T2 on boss defeat, T3 on island boss (Main only). Fires OnMainPathSelected, OnSecondaryPathSelected, OnPathTierUp events.
- **Acceptance:**
  - [x] Main + Secondary path selection logic
  - [x] 3rd path enforced as locked
  - [x] Tier progression triggers
  - [x] T3 only for Main path
  - [x] Events fired on selection and tier-up
  - [x] IPathProvider interface implemented

### T019: PathSelectionUI [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T018
- **Files:** `Roguelite/PathSelectionUI.cs`
- **Description:** Upgrade shrine UI. Shows 3 path options (or 2 for secondary). Displays path name, description, T1 ability preview, stat bonuses. Locks selected path, greys out for secondary selection. Confirmation button.
- **Acceptance:**
  - [ ] Shows all available paths for character
  - [ ] Displays stat bonuses and ability preview
  - [ ] Handles both Main and Secondary selection states
  - [ ] Confirmation before locking

### T020: RitualData ScriptableObject + Families [DONE]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T001
- **Files:** `Roguelite/RitualData.cs`, `Editor/RitualDataCreator.cs`
- **Branch:** `pillar2/T020-ritual-data-so` | **Merged:** 2026-03-03 → `gal`
- **Description:** RitualData SO: name, family, category, secondFamily (Twin only), maxLevel, basePower per level, trigger, effect type, effect prefab. Create initial Fire and Lightning family rituals (4-5 each).
- **Acceptance:**
  - [x] RitualData SO with all fields
  - [x] Fire family: Burn, Blazing Dash, Flame Strike, Ember Shield
  - [x] Lightning family: Chain Lightning, Lightning Strike, Shock Wave, Static Field
  - [x] Level 1-3 power values defined

### T021: RitualSystem — Trigger Pipeline [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T020
- **Files:** `Roguelite/RitualSystem.cs`
- **Description:** Manages active rituals per run. Subscribes to ICombatEvents. When combat event fires, checks all active rituals for matching trigger, fires ritual effects. Tracks family counts. Supports adding/leveling rituals. Provides IBuffProvider queries.
- **Acceptance:**
  - [ ] Subscribes to all ICombatEvents triggers
  - [ ] Trigger → matching ritual → fire effect pipeline
  - [ ] Add ritual (new or level-up existing)
  - [ ] Family count tracking
  - [ ] IBuffProvider implementation for damage/speed/defense multipliers

### T022: BasicEnemyAI [PENDING]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 3 | **Depends on:** T011
- **Files:** `World/EnemyAI.cs`, `World/EnemyStateBase.cs`, `World/States/IdleState.cs`, `PatrolState.cs`, `ChaseState.cs`, `AttackState.cs`, `HitReactState.cs`, `DeathState.cs`
- **Description:** State machine: Idle → Patrol → Chase → Attack → HitReact → Death. Configurable aggression, attack frequency, telegraph duration. Uses AttackData SOs for attack patterns. Respects TelegraphType for visual signals.
- **Acceptance:**
  - [ ] 6-state machine with clean transitions
  - [ ] Configurable per-enemy: aggression, frequency, range
  - [ ] Uses AttackData for attack patterns
  - [ ] Telegraph visuals (normal vs red/unstoppable)
  - [ ] Inherits from EnemyBase

### T023: Enemy Attack Patterns + Telegraphs [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 3 | **Depends on:** T022, T005
- **Files:** `World/EnemyAttackPatterns.cs`, `ScriptableObjects/Enemies/BasicEnemy.asset`
- **Description:** Attack pattern system using AttackData sequences. Telegraph visual system: Normal = wind-up animation, Unstoppable = red flash overlay. Create basic enemy type with 2-3 attack patterns.
- **Acceptance:**
  - [ ] Attack sequences from ScriptableObjects
  - [ ] Visual telegraph: normal vs red flash
  - [ ] Basic enemy with 2-3 attack patterns
  - [ ] Configurable telegraph duration

### T024: Character Animator Controllers [DONE]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 3 | **Depends on:** T014
- **Branch:** `shared/T024-animator-controllers` | **Completed:** 2026-03-05
- **Files:** `Editor/Animation/AnimationBuilder.cs`, `Editor/Animation/AnimationForgeMetadata.cs`, `Editor/Animation/AnimationEventStamper.cs`, `Scripts/Combat/Animation/TomatoFighterAnimatorParams.cs`
- **Description:** Base Animator Controller with shared state machine (idle, walk, run, 10 attack slots, dash, jump, land, block, guard, hurt, death). 4 Animator Override Controllers with character-specific clips and placeholder generation. AnimationEventStamper pipeline step for hitbox/combo events.
- **Acceptance:**
  - [x] Base controller with all combat states (locomotion, airborne, dash, 10 attack slots, defense, hurt, death)
  - [x] 4 override controllers (Brutor, Slasher, Mystica, Viper)
  - [x] Animation Events stamped on attack clips for hitbox, combo window, finisher timing
  - [x] Transition conditions match combat system states
  - [x] ERROR logged for metadata animations that don't map to any canonical state
  - [x] WARNING logged for canonical states with no matching animation (placeholder generated)

### T025: HUD — Health, Mana, Combo Counter [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 3 | **Depends on:** T007
- **Files:** `World/UI/HUDManager.cs`, `World/UI/HealthBarUI.cs`, `World/UI/ManaBarUI.cs`, `World/UI/ComboCounterUI.cs`
- **Description:** Screen-space overlay HUD. Health bar reads from character stats. Mana bar for MNA. Combo counter tracks consecutive hits with decay timer. Path indicator shows current Main/Secondary. Enemy health bars as world-space canvas.
- **Acceptance:**
  - [ ] Player health bar (% of max HP)
  - [ ] Mana bar
  - [ ] Combo counter with decay
  - [ ] Path indicator (icons for Main/Secondary)
  - [ ] Enemy world-space health bars

---

## Phase 3: Defensive Depth + Build Crafting
> Status: IN_PROGRESS | Tasks: 2/9 | Weeks: 5-6
> Goal: Full loop — fight area, pick ritual, select path at shrine, fight boss

### T026: PressureSystem + Stun [DONE]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T016
- **Branch:** `combat/T026-pressure-system-stun` | **Completed:** 2026-03-04
- **Files:** `Shared/Data/DamagePacket.cs`, `Shared/Data/CombatEventData.cs`, `Shared/Interfaces/ICombatEvents.cs`, `Combat/Hitbox/HitboxManager.cs`, `World/EnemyBase.cs`, `Combat/PlayerDamageable.cs`
- **Description:** Wire StunRate into damage pipeline. DamagePacket carries pre-calculated stunFillAmount (attacker-side). HitboxManager calculates stunFill = damage × stunRate × punishMultiplier. EnemyBase uses packet.stunFillAmount instead of recalculating. StunTriggered/StunRecovered events fired for Camera/UI/Roguelite integration.
- **Acceptance:**
  - [x] Pressure meter per enemy
  - [x] Punish multiplier (2x fill rate)
  - [x] Stun state with configurable duration
  - [x] Invulnerable recovery after stun
  - [x] Camera zoom event on stun
  - [x] PRS stat integration

### T027: WallBounce + AirJuggle [DONE]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 1 | **Depends on:** T015
- **Branch:** `combat/T027-wallbounce-airjuggle` | **Completed:** 2026-03-04
- **Files:** `Combat/Juggle/WallBounceHandler.cs`, `Combat/Juggle/JuggleSystem.cs`, `Shared/Enums/JuggleState.cs`, `Shared/Interfaces/IJuggleTarget.cs`, `Shared/Data/JuggleConfig.cs`
- **Description:** Wall bounce: detect wall collision during knockback → bounce. Unlimited per combo, minor damage, no extra pressure. Air juggle: track airborne state, Gale element extends airtime. After stun expires → invulnerable landing. OTG vs Tech Hit distinction.
- **Acceptance:**
  - [x] Wall bounce detection and physics
  - [x] Unlimited bounces per combo
  - [x] Airborne state tracking
  - [x] OTG vs Tech Hit distinction
  - [x] Gale extension support (via IBuffProvider)

### T028: Path T1 Ability Execution [PENDING]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T017, T018
- **Files:** `Characters/PathAbilityExecutor.cs`, `Characters/Abilities/Warden/Provoke.cs`, `Bulwark/IronGuard.cs`, `Guardian/ShieldLink.cs`, `Executioner/MarkForDeath.cs`, `Reaper/CleavingStrikes.cs`, `Shadow/PhaseDash.cs`, `Sage/MendingAura.cs`, `Enchanter/Empower.cs`, `Conjurer/SummonSproutling.cs`, `Marksman/PiercingShots.cs`, `Trapper/HarpoonShot.cs`, `Arcanist/ManaCharge.cs`
- **Description:** Execute all 12 Tier 1 path abilities through the combat system. Each ability reads from PathData SO, checks tier via IPathProvider, consumes mana where applicable. Active abilities have cooldowns. Passive abilities modify combat calculations.
- **Acceptance:**
  - [ ] All 12 T1 abilities functional
  - [ ] Cooldown management per ability
  - [ ] Mana consumption where specified
  - [ ] Integrates with IPathProvider tier checks
  - [ ] Fires PathAbilityEventData through ICombatEvents

### T029: RitualStackCalculator [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T021
- **Files:** `Roguelite/RitualStackCalculator.cs`
- **Description:** Plain C# class. Level scaling: L1=base, L2=1.5x, L3=2.0x. Same-mechanic stacking: 2 rituals=1.5x, 3=2.0x. Ritual Power from trinkets is multiplicative with itself (1.1 * 1.1 = 1.21x). Final = (base * levelMult) * stackingMult * ritualPower.
- **Acceptance:**
  - [ ] Level multipliers: 1.0, 1.5, 2.0
  - [ ] Stacking multipliers: 1.0, 1.5, 2.0
  - [ ] Ritual Power multiplicative (NOT additive)
  - [ ] Unit tests with concrete numeric examples
  - [ ] Plain C# class for testability

### T030: TrinketSystem [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T007
- **Files:** `Roguelite/TrinketSystem.cs`, `Shared/Data/TrinketData.cs`
- **Description:** Stat modifier items found during runs. TrinketData SO: stat type, modifier value, modifier type (flat/percent), condition (always/on-dodge/on-kill/etc.). TrinketSystem manages active trinkets, applies modifiers through CharacterStatCalculator.
- **Acceptance:**
  - [ ] TrinketData SO with all fields
  - [ ] Flat and percentage modifiers
  - [ ] Conditional triggers
  - [ ] Integration with CharacterStatCalculator

### T031: RewardSelectorUI [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T021
- **Files:** `Roguelite/RewardSelectorUI.cs`
- **Description:** Post-area reward selection. Pick 1 of 2 rituals (3 with Soul Tree upgrade), OR gold, OR crystals. Shows ritual name, family, description, preview of effect. Fires event on selection.
- **Acceptance:**
  - [ ] 2-3 ritual options displayed
  - [ ] Gold and crystal alternatives
  - [ ] Ritual preview with family color coding
  - [ ] Selection fires event consumed by RitualSystem

### T032: BossAI Framework [PENDING]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 3 | **Depends on:** T022, T026
- **Files:** `World/BossAI.cs`, `World/BossPhase.cs`, `ScriptableObjects/Enemies/TestBoss.asset`
- **Description:** Extends EnemyAI with Phase system. Phases transition by HP% thresholds. Each phase: own attack pattern list (SO), new moves, faster tempo. Punish windows after big attacks (configurable duration). Camera zoom on stun. Create one test boss.
- **Acceptance:**
  - [ ] Phase system with HP% transitions
  - [ ] Per-phase attack patterns (SO)
  - [ ] Punish windows after big attacks
  - [ ] Camera zoom on stun (event-based)
  - [ ] One test boss with 2-3 phases

### T033: Branching Path Navigation [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 3 | **Depends on:** T010
- **Files:** `World/PathNavigator.cs`, `World/PathNodeData.cs`, `World/BranchingPathUI.cs`
- **Description:** Map UI showing route options. PathNodeData SO: node type (Combat, Shop, Challenge, Boss, Inspiration, Story), connections to next nodes. Fork choices at area transitions. Different routes offer different rewards/challenges.
- **Acceptance:**
  - [ ] PathNodeData SO with node type and connections
  - [ ] Map UI visualization
  - [ ] Fork choice interaction
  - [ ] Node type icons/colors

### T034: Path T1 Ability VFX [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 3 | **Depends on:** T028
- **Files:** `Prefabs/Effects/Abilities/*.prefab`
- **Description:** Visual effects for all 12 T1 path abilities. Brutor: taunt pulse, shield glow, link beam. Slasher: mark indicator, cleave arc, afterimage trail. Mystica: heal particles, buff glow, sproutling spawn. Viper: pierce trail, harpoon chain, charge buildup.
- **Acceptance:**
  - [ ] 12 VFX prefabs (one per T1 ability)
  - [ ] Particle systems + sprite effects
  - [ ] Color-coded per character
  - [ ] Performance: max 50 particles per effect

---

## Phase 4: Advanced Combat + Meta-Progression
> Status: PENDING | Tasks: 0/10 | Weeks: 7-8
> Goal: Complete run from hub → 2 areas → boss → hub with path progress saved

### T035: RepetitiveTracker — Anti-Spam [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 1 | **Depends on:** T014
- **Files:** `Combat/RepetitiveTracker.cs`
- **Description:** Track consecutive same-move usage. After threshold (configurable, ~3) → damage penalty multiplier (0.5x). Resets on using different move. IBuffProvider can override (Enchanter buff).
- **Acceptance:**
  - [ ] Tracks consecutive same-move count
  - [ ] Configurable threshold and penalty
  - [ ] Reset on move variety
  - [ ] Override check via IBuffProvider

### T036: OTG vs TechHit System [PENDING]
- **Type:** implementation | **Priority:** P2 | **Owner:** Dev 1 | **Depends on:** T027
- **Files:** `Combat/OTGSystem.cs`
- **Description:** Tech Hit: hitting downed enemies — they can recover. OTG: Arcana hits downed enemies, they cannot recover. Distinction critical for combo design. State tracking: airborne → grounded → downed → recovering.
- **Acceptance:**
  - [ ] Downed state tracking
  - [ ] Tech Hit recovery window
  - [ ] Arcana bypass (OTG)
  - [ ] State machine: airborne/grounded/downed/recovering

### T037: Path T2 + T3 Ability Execution [PENDING]
- **Type:** implementation | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T028
- **Files:** `Characters/Abilities/*/` (T2 and T3 for all 12 paths)
- **Description:** Implement all T2 abilities (12) and T3 signature abilities (12). T2: enhanced versions (Aggro Aura, Retaliation, Rallying Presence, Execution Threshold, Chain Slash, Afterimage, etc.). T3: capstones (Wrath, Fortress, Aegis Dome, Deathblow, Whirlwind, Thousand Cuts, etc.).
- **Acceptance:**
  - [ ] All 12 T2 abilities functional
  - [ ] All 12 T3 abilities functional
  - [ ] T3 gated by Main path check
  - [ ] Cooldowns, mana costs, durations per spec

### T038: MetaProgression + SoulTree [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T009
- **Files:** `Roguelite/MetaProgression.cs`, `Roguelite/SoulTree.cs`
- **Description:** Persistent between runs. Soul Tree: health+, damage+, self-revives, rare chance, 3rd ritual choice. Currencies: Crystals (carry over), Imbued Fruits, Primordial Seeds. Unlock nodes with crystals. Modular tree — not rigid branching.
- **Acceptance:**
  - [ ] Soul Tree with unlockable nodes
  - [ ] Crystal cost per node
  - [ ] Stat bonuses applied through CharacterStatCalculator
  - [ ] 3rd ritual choice upgrade
  - [ ] Persistence between runs

### T039: SaveSystem [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T038
- **Files:** `Roguelite/SaveSystem.cs`, `Shared/Data/MetaProgressionData.cs`
- **Description:** JSON serialization to Application.persistentDataPath. Stores: MetaProgressionData (currencies, soul tree, unlocks), per-character unlocks (arcanas, inspirations), quest state, run history.
- **Acceptance:**
  - [ ] JSON save/load to persistentDataPath
  - [ ] MetaProgressionData fully serialized
  - [ ] Per-character unlock tracking
  - [ ] Save versioning for future migrations

### T040: HubManager — Between-Run Base [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T038, T039
- **Files:** `Roguelite/HubManager.cs`
- **Description:** The between-run hub area. Character selection (4 characters with stat preview). Arcana selection. Soul Tree access. NPC interactions (unlock rituals, inspirations). Shop. Quest board. Loads save data on enter.
- **Acceptance:**
  - [ ] Character selection with stat display
  - [ ] Soul Tree UI
  - [ ] NPC interaction framework
  - [ ] Save data loaded on hub enter

### T041: InspirationSystem [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T018
- **Files:** `Roguelite/InspirationSystem.cs`, `Shared/Data/InspirationData.cs`, `ScriptableObjects/Inspirations/**`
- **Description:** Character-specific move unlocks. InspirationData SO: character, path, description, modifier type. 24 total (6 per character, 2 per path). Dropped from minibosses. Some permanently unlockable via Seeds.
- **Acceptance:**
  - [ ] InspirationData SO with all fields
  - [ ] 24 inspiration assets created
  - [ ] Drop system from miniboss events
  - [ ] Permanent unlock via Primordial Seeds

### T042: 2nd Enemy Type + Mini-Bosses [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 3 | **Depends on:** T022, T032
- **Files:** `World/Enemies/HeavyEnemy.cs`, `World/Enemies/MiniBoss.cs`, `ScriptableObjects/Enemies/Heavy.asset`, `MiniBoss.asset`
- **Description:** Heavy enemy: slow, Unstoppable attacks, high HP. Mini-boss: 2-phase simplified boss, drops Inspirations. Both inherit EnemyBase. Different attack patterns, telegraphs.
- **Acceptance:**
  - [ ] Heavy enemy with Unstoppable attacks
  - [ ] Mini-boss with 2 phases
  - [ ] Mini-boss drops Inspiration event
  - [ ] Distinct visual telegraphs per enemy

### T043: Quest System [PENDING]
- **Type:** implementation | **Priority:** P2 | **Owner:** Dev 3 | **Depends on:** T033
- **Files:** `World/QuestSystem.cs`, `Shared/Data/QuestData.cs`
- **Description:** Side quests tracked on map. QuestData SO: trigger, completion condition, reward, world state change. Quest markers on branching path map. World changes based on quest completion.
- **Acceptance:**
  - [ ] QuestData SO
  - [ ] Quest tracking + map markers
  - [ ] Completion conditions
  - [ ] World state changes

### T044: Path T2 + T3 Ability VFX [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 3 | **Depends on:** T037
- **Files:** `Prefabs/Effects/Abilities/T2/*.prefab`, `T3/*.prefab`
- **Description:** VFX for all T2 (12) and T3 (12) abilities. T3 signature abilities need dramatic effects: Wrath fire aura, Fortress shockwave, Aegis Dome shield bubble, Deathblow execution slash, Whirlwind tornado, Thousand Cuts teleport trails, Resurrection light pillar, etc.
- **Acceptance:**
  - [ ] 12 T2 VFX prefabs
  - [ ] 12 T3 VFX prefabs (signature — dramatic)
  - [ ] Consistent per-character color themes
  - [ ] Performance budget respected

---

## Phase 5: Content + Co-op
> Status: PENDING | Tasks: 0/8 | Weeks: 9-10
> Goal: All 4 characters polished, 2 islands playable, co-op on couch

### T045: Character Combo Feel Pass [PENDING]
- **Type:** refactor | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T037
- **Files:** All `Combat/*.cs`, `Characters/*.cs`, character AttackData SOs
- **Description:** Tune all 4 characters to feel distinct. Brutor: heavy, slow recovery. Slasher: snappy, tight cancels. Mystica: floaty, smooth transitions. Viper: sharp, precise. Adjust hitstop, screen shake, timing values.
- **Acceptance:**
  - [ ] Each character feels mechanically distinct
  - [ ] Hitstop tuned per attack weight
  - [ ] Screen shake per impact
  - [ ] Cancel windows feel responsive

### T046: Arcana System [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 1 | **Depends on:** T037
- **Files:** `Combat/ArcanaSystem.cs`, `Shared/Data/ArcanaData.cs`
- **Description:** Mana-consuming special abilities. Selected before run. One Arcana glows = bonus XP. Second Arcana unlocked mid-run after Island 1. Per-character arcana options (3-4 each). Unlocked via Primordial Seeds.
- **Acceptance:**
  - [ ] ArcanaData SO with mana cost, effect, cooldown
  - [ ] Pre-run selection
  - [ ] 2nd arcana unlock mid-run
  - [ ] Per-character arcana options

### T047: Twin Rituals [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T029
- **Files:** `Roguelite/TwinRitualSystem.cs`, `ScriptableObjects/Rituals/Twins/*.asset`
- **Description:** Cross-family ritual combinations. Require both families present. Fire+Lightning = Thunder Burn. Fire+Thorn = Burning Brambles. Create 4-6 Twin Ritual assets. Unlockable via hub NPC.
- **Acceptance:**
  - [ ] Twin rituals require both families
  - [ ] 4-6 twin ritual assets
  - [ ] Unlock via hub NPC
  - [ ] Stack with regular rituals

### T048: All 8 Ritual Families Complete [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T029
- **Files:** `ScriptableObjects/Rituals/{Water,Thorn,Gale,Time,Cosmic,Necro}/*.asset`
- **Description:** Complete remaining 6 families (Fire + Lightning done in T020). Water: tidal waves. Thorn: bramble knives. Gale: extended juggles. Time: echoes. Cosmic: ultimates. Necro: lifesteal/summons. 4-5 rituals per family.
- **Acceptance:**
  - [ ] All 8 families with 4-5 rituals each
  - [ ] Core, General, Enhancement categories per family
  - [ ] All trigger-effect links defined
  - [ ] Level 1-3 power values balanced

### T049: Ritual Balance Pass [PENDING]
- **Type:** refactor | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T048
- **Files:** All ritual ScriptableObject assets
- **Description:** Test synergy combinations. No single ritual should be strictly dominant. Twin rituals should feel rewarding but not required. Path-ritual interactions should create interesting choices (not obvious best combos).
- **Acceptance:**
  - [ ] No strictly dominant ritual
  - [ ] Path-ritual interactions tested
  - [ ] Twin ritual power level appropriate
  - [ ] Documented balance notes

### T050: 2nd Island [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 3 | **Depends on:** T042
- **Files:** `Scenes/Islands/Island2/`, new enemy types, boss
- **Description:** Second island with unique: enemy set (2-3 new types), sub-bosses, environment, boss. Different difficulty level. New branching path options.
- **Acceptance:**
  - [ ] 2-3 new enemy types
  - [ ] New boss with 3 phases
  - [ ] Unique environment visual
  - [ ] Branching paths with new node types

### T051: Local Co-op [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 3 | **Depends on:** T012
- **Files:** `World/CoopManager.cs`, camera framing updates
- **Description:** 2-player local couch co-op. Split input (Player 1 + Player 2 controllers). Camera frames both players. Separate builds (each player selects character, paths, rituals independently). Players don't share gold.
- **Acceptance:**
  - [ ] 2-player input split
  - [ ] Camera frames both players
  - [ ] Independent character/path/ritual selection
  - [ ] Separate currency tracking

### T052: Mounts & Companions [PENDING]
- **Type:** implementation | **Priority:** P2 | **Owner:** Dev 3 | **Depends on:** T022
- **Files:** `World/MountSystem.cs`, `World/CompanionAI.cs`
- **Description:** Rideable mounts in certain areas. AI mercenaries hireable. Both have combat capabilities. Companion AI follows player, auto-attacks nearest enemy.
- **Acceptance:**
  - [ ] Mount riding mechanics
  - [ ] Companion AI (follow + auto-attack)
  - [ ] Combat capabilities for both
  - [ ] Purchase/hire flow

---

## Phase 6: Polish + Full Loop
> Status: PENDING | Tasks: 0/8 | Weeks: 11-12
> Goal: Full playable vertical slice — hub → 3 islands → final boss

### T053: Game Feel Pass [PENDING]
- **Type:** refactor | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T045
- **Files:** All combat files, DOTween integration
- **Description:** Hitstop (2-3 frames on heavy), screen shake (per impact weight), speed lines on dash, impact particles, damage numbers, death slowmo. DOTween for smooth easing. Per-character feel identity.
- **Acceptance:**
  - [ ] Hitstop on all attacks (weighted)
  - [ ] Screen shake on impacts
  - [ ] Speed lines and particles
  - [ ] Death slowmo + camera zoom
  - [ ] Each character has distinct feel

### T054: Combat Balancing [PENDING]
- **Type:** refactor | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T053
- **Files:** All AttackData and PathData SOs
- **Description:** Balance all 4 characters. No character should be strictly better. Each should excel in their role. Path combinations should all be viable (6 per character = 24 total builds). DPS testing against standard enemy waves.
- **Acceptance:**
  - [ ] All 4 characters balanced
  - [ ] All 24 builds viable
  - [ ] DPS benchmarks documented
  - [ ] Difficulty progression smooth

### T055: Build Balance + Economy [PENDING]
- **Type:** refactor | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T049
- **Files:** All roguelite SOs, currency values
- **Description:** Tune currency drop rates, ritual power levels, trinket values. Soul Tree progression should feel meaningful but not required. Full run economy: enough crystals for 1-2 soul tree nodes per run.
- **Acceptance:**
  - [ ] Currency drop rates balanced
  - [ ] Soul Tree progression pacing
  - [ ] Ritual power levels final
  - [ ] Trinket value balance

### T056: Difficulty Scaling [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T055
- **Files:** `Roguelite/DifficultyScaling.cs`
- **Description:** Progressive difficulty across islands. Enemy HP/damage scales. More Unstoppable attacks in later islands. Boss phase speed increases. New Game+ mode after first clear.
- **Acceptance:**
  - [ ] Per-island difficulty multipliers
  - [ ] Enemy stat scaling
  - [ ] Boss tempo increase
  - [ ] NG+ mode with higher multipliers

### T057: Islands 3 + 4 [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 3 | **Depends on:** T050
- **Files:** `Scenes/Islands/Island3/`, `Island4/`
- **Description:** Complete island roster. 4 islands, player traverses 3 per run. Each unique: enemies, boss, environment, culture. Fork choices at island transitions.
- **Acceptance:**
  - [ ] 2 new islands with unique enemies
  - [ ] 2 new bosses (3 phases each)
  - [ ] Island selection at forks
  - [ ] 4-island rotation (play 3)

### T058: VFX + Sound Integration [PENDING]
- **Type:** implementation | **Priority:** P1 | **Owner:** Dev 3 | **Depends on:** T044
- **Files:** VFX prefabs, audio clips, AudioManager
- **Description:** Full VFX pass: hit sparks, element effects (fire/lightning/etc.), ritual activation, path ability visuals. Sound: hit impacts, UI clicks, ambient, boss music. All triggered via Animation Events.
- **Acceptance:**
  - [ ] Hit spark effects per damage type
  - [ ] Elemental VFX per ritual family
  - [ ] Sound effects for all combat actions
  - [ ] Music system (combat, boss, hub)

### T059: Online Co-op [PENDING]
- **Type:** implementation | **Priority:** P2 | **Owner:** Dev 3 | **Depends on:** T051
- **Files:** Network integration files
- **Description:** Online 2-player co-op using Fishnet or Mirror. Rollback netcode for combat precision. State sync for rituals, paths, enemies. Lobby system.
- **Acceptance:**
  - [ ] Online lobby + matchmaking
  - [ ] State sync (combat, rituals, paths)
  - [ ] Rollback netcode for combat
  - [ ] Playable at reasonable latency

### T060: Full Integration Test + Playtest [PENDING]
- **Type:** test | **Priority:** P0 | **Owner:** ALL | **Depends on:** all above
- **Files:** N/A
- **Description:** Full vertical slice playtest. Hub → character select → 3 islands → final boss → return to hub with progress. All 4 characters, all path combinations tested. Co-op tested. Balance issues logged.
- **Acceptance:**
  - [ ] Full run completable with all 4 characters
  - [ ] All 24 builds tested
  - [ ] Co-op stable
  - [ ] No critical bugs
  - [ ] Balance notes documented
  - [ ] Performance acceptable (60fps target)
