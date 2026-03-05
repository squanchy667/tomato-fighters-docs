# Changelog

## [Phase 1] — 2026-03-05 (T013 Basic Test Scene — DONE)

### Completed
- **T013**: Basic Test Scene — full integration of all Phase 1 systems into `MovementTest.unity`
  - **Art layer upgrade**: Replaced programmatic grid-line background with 6 forest environment sprites (`bg_forest_distant`, `bg_forest_midground`, `bg_forest_foreground`, `ground_forest_floor`, `wall_left_stone`, `wall_right_stone`). Auto-scaled via `sprite.bounds.size`.
  - **CameraController2D integration** (T012): Added to camera with smooth follow, arena bounds, SO event channels (`OnCameraLock`, `OnCameraUnlock`, `OnStunTriggered`, `OnStunRecovered`). Player transform wired as follow target.
  - **WaveManager integration** (T010): Created with 3 spawn points, 1 test wave (3 enemy spawns), all 6 SO event channels wired.
  - **LevelBound triggers**: Left/right trigger colliders inside arena walls, wired to `OnBoundReached` event channel.
  - **Floor constraint**: Walkable area constrained to floor sprite strip (y=-5 to y=-3). Top wall caps vertical movement. All entity spawn positions adjusted to floor.
  - **Layer collision fix**: Enabled `Default↔PlayerHurtbox` and `Default↔EnemyHurtbox` so arena walls block player and enemy bodies.

### Design Decisions
- **DD-4**: Upgraded `CreateArenaBackground()` with art layer sprites instead of programmatic rectangles
- **DD-5**: Background layers span full arena, stacked by sorting order (parallax deferred to T012 camera)
- **DD-6**: Stone wall sprites are visual-only; physics stays on invisible BoxCollider2D walls
- **DD-7**: Sprite scaling from native texture size via `sprite.bounds.size` — auto-adapts to any PPU
- Walkable area matches floor sprite height — characters move within the visual floor strip

## [Phase 2] — 2026-03-05 (T024 Character Animator Controllers — DONE)

### Completed
- **T024**: Character Animator Controllers — Base + Override architecture for all 4 characters
  - Base controller at `Animations/Base/BaseCharacter_Controller.controller` with full state machine: locomotion (idle/walk/run), airborne (jump/land), dash, 10 attack slots (attack_1–attack_10), defense (block/guard), reaction (hurt/death)
  - 4 AnimatorOverrideControllers: Mystica, Slasher, Brutor, Viper — character-specific clips mapped to shared state machine
  - AnimationEventStamper (Step 3 pipeline): stamps ActivateHitbox, DeactivateHitbox, OnComboWindowOpen, OnFinisherEnd from AttackData SO timing
  - Placeholder clip generation for canonical states without real art (single-frame idle sprite)
  - Validation: ERROR on unmapped metadata animations, WARNING on missing canonical states
  - TomatoFighterAnimatorParams updated: attack_1Trigger–attack_10Trigger, blockTrigger, guardTrigger, all state name constants
  - Character creator scripts updated to load override controllers instead of standalone controllers
  - CharacterPrefabConfig.animatorController changed from AnimatorController to RuntimeAnimatorController

### Design Decisions
- **DD-1**: Base + Override pattern — all characters share one state machine, only clips differ
- **DD-2**: Placeholder clips auto-generated from idle sprite for missing art
- **DD-3**: 10 generic attack slots (attack_1–attack_10), no semantic naming at controller level
- **DD-4**: AnimationEventStamper as separate pipeline Step 3 (decoupled from AnimationBuilder)
- **DD-5**: Base controller uses Mystica's clips as template
- **DD-6**: Defense + reaction states (block, guard, hurt, death) with trigger-driven transitions

### Notes
- ComboDefinition animationTrigger strings still use old naming (attack_light_1, etc.) — need updating to attack_1–attack_10 in a follow-up
- Pipeline order: Import Sprite Sheets → Build Animations → Stamp Animation Events
- Guard state is looping (stays until interrupted by another trigger via AnyState)

## [Phase 1] — 2026-03-05 (T010 WaveManager + T012 CameraController2D — DONE)

### Completed
- **T012: CameraController2D** — branch `ofek`
  - Smooth side-scrolling camera with configurable leading, smooth damping, and vertical offset
  - Dynamic level bounds via `SetBounds()` with Gizmo visualization
  - SO event channel integration: camera lock/unlock (WaveManager), stun zoom (PressureSystem)
  - Co-op framing: multi-target center calculation with auto-zoom to frame all players
  - `SnapToTarget()` for instant positioning on scene transitions
  - `AddTarget()`/`RemoveTarget()` API for co-op player join/leave
  - Unblocks: T013 (Test Scene), T051 (Local Co-op)

- **T010: WaveManager** — branch `ofek` (commit `8a9d6ea`)
  - `WaveManager` MonoBehaviour (316 LOC): Configurable wave spawning, camera stops at LevelBounds until wave cleared, fires events on wave start/clear/area complete
  - `LevelBound` MonoBehaviour: Camera boundary trigger zones, stops camera scroll until wave is cleared
  - `WaveData` ScriptableObject: Wave composition config (enemy type, count, spawn delay)
  - `EnemySpawnData` struct: Per-enemy spawn definition (prefab, position offset, delay)
  - `VoidEventChannel` + `IntEventChannel` SO event channels: Cross-pillar event system (OnWaveStart, OnWaveCleared, OnAreaComplete)
  - `WaveManagerAssetsCreator` editor script: Menu-driven SO asset creation
  - Unblocks: T013 (Test Scene), T033 (Branching Path Navigation)

## [Phase 3] — 2026-03-04 (T027 WallBounce + AirJuggle — DONE)

### Completed
- **T027: WallBounce + AirJuggle** — branch `combat/T027-wallbounce-airjuggle` (commit `8769d7a`)
  - `WallBounceHandler` MonoBehaviour: Detects wall collisions via `OnCollisionEnter2D`, reflects velocity with configurable retention factor, fires `BounceDetected` event. Unlimited bounces per combo. Minor damage (no pressure fill) applied via `IJuggleTarget.OnWallBounced` → `EnemyBase.HandleWallBounce()`
  - `JuggleSystem` MonoBehaviour implementing `IJuggleTarget`: Belt-scroll simulated height tracking (mirrors CharacterMotor's jump model). 5-state lifecycle: Grounded → Airborne → Falling → OTG → TechRecover → Grounded. Queries `IBuffProvider.GetJuggleGravityMultiplier()` for Gale element airtime extension
  - `JuggleConfig` ScriptableObject: Tuning params for gravity (25 u/s²), terminal fall speed (20), bounce velocity retention (0.7), min bounce velocity (3), wall bounce damage (2), OTG duration (1.0s), tech recover duration (0.4s), knockback recovery time (0.5s)
  - `JuggleState` enum in `Shared/Enums/`: Grounded, Airborne, Falling, OTG, TechRecover
  - `IJuggleTarget` interface in `Shared/Interfaces/`: Cross-pillar contract — Combat implements, World queries via `GetComponent<IJuggleTarget>()`
  - `IBuffProvider.GetJuggleGravityMultiplier()`: New interface method for Gale element gravity reduction (base 1.0, lower = slower fall)
  - `WallBounceEventData` + `JuggleLandEventData` structs added to `CombatEventData.cs`
  - `EnemyBase` integration: IJuggleTarget wired in `Awake()`, `ApplyKnockback()` notifies juggle system, `ApplyLaunch()` delegates to IJuggleTarget, `StunRoutine()` defers invulnerability blink until landing if airborne
  - Unblocks: T036 (OTG vs TechHit System)

### Design Decisions
- Belt-scroll simulated height (not Rigidbody2D Y-axis) — consistent with CharacterMotor's jump model; `spriteTransform.localPosition.y = airHeight`
- IJuggleTarget in Shared/Interfaces/ for cross-pillar access — Combat implements, World queries without pillar violation
- WallBounceHandler uses velocity magnitude + `IsInKnockback` flag as dual guard — prevents bouncing during normal movement
- Wall bounce damage bypasses TakeDamage pipeline (no re-trigger of knockback) — applied directly to health via HandleWallBounce
- JuggleSystem manages knockback recovery timing (replacing EnemyBase coroutine when present) — single source of truth for velocity state
- OTG → TechRecover → Grounded gives clear states for T036 to distinguish OTG hits vs tech recovery

### Files Added
- `Scripts/Shared/Enums/JuggleState.cs`
- `Scripts/Shared/Interfaces/IJuggleTarget.cs`
- `Scripts/Shared/Data/JuggleConfig.cs`
- `Scripts/Combat/Juggle/JuggleSystem.cs`
- `Scripts/Combat/Juggle/WallBounceHandler.cs`

### Files Modified
- `Scripts/Shared/Interfaces/IBuffProvider.cs` (+`GetJuggleGravityMultiplier()`)
- `Scripts/Shared/Data/CombatEventData.cs` (+`WallBounceEventData`, +`JuggleLandEventData`)
- `Scripts/World/EnemyBase.cs` (IJuggleTarget integration, knockback/launch delegation, deferred post-stun invuln)

### Notes
- Task counter: 18/60 (Phase 1: 10/13, Phase 2: 6/12, Phase 3: 2/9)
- T027 unblocks: T036 (OTG vs TechHit System)
- No existing tests affected — IBuffProvider has no implementations yet
- JuggleConfig needs a `.asset` created in Unity and assigned to enemy prefabs via Creator Scripts

---

## [Phase 3] — 2026-03-04 (T026 PressureSystem + Stun — DONE)

### Completed
- **T026: PressureSystem + Stun** — branch `combat/T026-pressure-system-stun` (commit `1727919`)
  - `DamagePacket.stunFillAmount` — new readonly field carrying pre-calculated pressure fill from attacker stats (DD-1: attacker-side calculation)
  - `HitboxManager.stunRate` — placeholder `[SerializeField]` field (default 1.0) with per-character values documented in tooltip (Slasher=1.5, Brutor=1.0, Viper=0.8, Mystica=0.5)
  - `BuildDamagePacket()` now calculates `stunFill = damage * stunRate * (isPunish ? 2f : 1f)` and bakes it into the packet
  - `EnemyBase.TakeDamage()` uses `packet.stunFillAmount` instead of recalculating internally — defender just receives "how much pressure this hit applies"
  - `EnemyBase._lastHitBy` tracks source `CharacterType` for event data
  - `StunTriggered` / `StunRecovered` events on `EnemyBase` — fired from `StunRoutine()` for Camera/UI/Roguelite subscribers
  - `StunEventData` (lastHitBy, stunnedPosition, stunDuration) + `StunRecoveredEventData` (recoveredPosition) in `CombatEventData.cs`
  - `ICombatEvents.OnStun` + `OnStunRecovered` — cross-pillar event signatures for Roguelite integration
  - `PlayerDamageable.AddStun()` remains stub with TODO comment (DD-3: player stun deferred)
  - Default parameter `stunFillAmount = 0f` on `DamagePacket` constructor — backward compatible with existing call sites (TestDummyEnemy)

### Design Decisions
- DD-1 (T026): StunFill calculated attacker-side — HitboxManager owns the calculation, EnemyBase just receives the amount
- DD-2 (T026): No pressureResistance field — use higher `pressureThreshold` for tanky enemies (one knob, not two)
- DD-3 (T026): Player stun deferred — requires input lock, combo cancel, defense reset, UI (separate task)
- DD-4 (T026): Camera zoom event only — fire `OnStun`, don't consume. Camera belongs to World pillar
- DD-5 (T026): StunRate as placeholder field — matches `baseAttack` placeholder pattern, both wired to stat system together later

### Files Modified
- `Scripts/Shared/Data/DamagePacket.cs` (added `stunFillAmount` field + constructor param)
- `Scripts/Shared/Data/CombatEventData.cs` (added `StunEventData`, `StunRecoveredEventData`)
- `Scripts/Shared/Interfaces/ICombatEvents.cs` (added `OnStun`, `OnStunRecovered`)
- `Scripts/Combat/Hitbox/HitboxManager.cs` (added `stunRate`, updated `BuildDamagePacket`)
- `Scripts/World/EnemyBase.cs` (uses `stunFillAmount`, tracks `_lastHitBy`, fires events)
- `Scripts/Combat/PlayerDamageable.cs` (updated TODO comment)

### Notes
- Task counter: 17/60 (Phase 1: 10/13, Phase 2: 6/12, Phase 3: 1/9)
- Phase 3 is now IN_PROGRESS
- T026 unblocks: T032 (BossAI Framework)

---

## [Phase 2] — 2026-03-04 (T017 Character Passives — DONE)

### Completed
- **T017: Character Passives** — branch `combat/T017-character-passives` (commit `f52f526`)
  - 4 passive abilities as plain C# classes with `PassiveConfig` ScriptableObject for tuning
  - **ThickSkin** (Brutor): Constant 0.85 defense multiplier (15% DR) + 0.6 knockback multiplier (40% reduction)
  - **Bloodlust** (Slasher): +3% ATK per hit landed, 10 stacks max (+30%), resets to 0 after 3s without a hit. Combo drops do NOT reset stacks
  - **ArcaneResonance** (Mystica): +5% damage per cast (multiplicative: 1.05^stacks), 3 max stacks with independent 3s expiry timers. Triggers on any attack event
  - **DistanceBonus** (Viper): +2% per unit distance at hit time, capped at +30% (15 units). Distance read from HitContext struct — no transform access
  - `IPassiveProvider` interface in `Shared/Interfaces/` — dedicated passive channel, separate from `IBuffProvider`
  - `HitContext` struct in `Shared/Data/` — per-hit context (damageType, distanceToTarget, isPunishHit)
  - `PassiveAbilitySystem` MonoBehaviour — holds active passive, subscribes to `HitboxManager.OnHitProcessed` + `ComboController.AttackStarted`, implements `IPassiveProvider`
  - All 4 character Creator Scripts updated to wire `PassiveAbilitySystem` + shared `PassiveConfig` SO
  - Unblocks: T028 (Path T1 Ability Execution)

### Design Decisions
- DD-1 (T017): PassiveConfig SO + plain C# logic — tunable values in Inspector, logic unit-testable without Unity
- DD-2 (T017): Self-buff only for Arcane Resonance — co-op doesn't exist yet, add ally broadcast later
- DD-3 (T017): HitContext struct for distance/per-hit data — keeps passives decoupled from transforms
- DD-4 (T017): IPassiveProvider separate from IBuffProvider — passives are Combat-internal, IBuffProvider is Roguelite's channel
- DD-5 (T017): Bloodlust decay timer only — combo drops do NOT reset stacks (double-punishment too harsh)

### Tests Added
- `ThickSkinTests.cs` — 6 tests (constant multipliers, custom config)
- `BloodlustTests.cs` — 10 tests (stacks, cap, decay, timer reset, sequential increment)
- `ArcaneResonanceTests.cs` — 11 tests (stacks, multiplicative calc, independent expiry, slot replacement)
- `DistanceBonusTests.cs` — 9 tests (linear scaling, cap, per-hit independence)

### Files Added
- `Scripts/Shared/Data/HitContext.cs`
- `Scripts/Shared/Interfaces/IPassiveProvider.cs`
- `Scripts/Characters/Passives/PassiveConfig.cs`
- `Scripts/Characters/Passives/IPassiveAbility.cs`
- `Scripts/Characters/Passives/ThickSkin.cs`
- `Scripts/Characters/Passives/Bloodlust.cs`
- `Scripts/Characters/Passives/ArcaneResonance.cs`
- `Scripts/Characters/Passives/DistanceBonus.cs`
- `Scripts/Characters/PassiveAbilitySystem.cs`
- `Tests/EditMode/Characters/ThickSkinTests.cs`
- `Tests/EditMode/Characters/BloodlustTests.cs`
- `Tests/EditMode/Characters/ArcaneResonanceTests.cs`
- `Tests/EditMode/Characters/DistanceBonusTests.cs`

### Files Modified
- `Editor/Prefabs/CharacterPrefabConfig.cs` (added `passiveConfig` field)
- `Editor/Prefabs/PlayerPrefabCreator.cs` (wires PassiveAbilitySystem)
- `Editor/Characters/BrutorCharacterCreator.cs` (loads/creates PassiveConfig)
- `Editor/Characters/SlasherCharacterCreator.cs` (loads/creates PassiveConfig)
- `Editor/Characters/MysticaCharacterCreator.cs` (loads/creates PassiveConfig)
- `Editor/Characters/ViperCharacterCreator.cs` (loads/creates PassiveConfig)
- `Tests/EditMode/TomatoFighters.Tests.EditMode.asmdef` (added Characters assembly ref)

### Notes
- Task counter: 16/60 (Phase 1: 10/13, Phase 2: 6/12)
- T017 unblocks: T028 (Path T1 Ability Execution)
- IPassiveProvider is not yet integrated into HitboxManager's damage calculation — that wiring happens when the full damage formula is assembled

---

## [Phase 2] — 2026-03-04 (T016 Follow-up: Clash Cancellation + Defense Visual Cues)

### Changes
- **Clash cancellation (0% mutual damage):** When two attacks collide simultaneously, both resolve as `DamageResponse.Clashed` with zero damage applied. `ClashTracker` component provides reciprocal immunity (0.1s window) so HitboxManager skips damage on the return frame
- **ClashTracker component** (`Shared/Components/ClashTracker.cs`): New MonoBehaviour tracking recent clash partners via `Dictionary<GameObject, float>`. `RegisterClash(target)` grants mutual immunity; `IsClashImmune(target)` queried by HitboxManager before damage resolution
- **Defense visual cues** (`Combat/Debug/DefenseDebugUI.cs`): Floating text feedback for Clash, Deflect, and Dodge outcomes. Font size 50, character size 0.12, color-coded per outcome. Attached to both player and enemy entities
- **NotifyDefenseSuccess** added to `IDefenseProvider` interface: Allows World pillar (TestDummyEnemy) to trigger visual feedback on the target's DefenseSystem after defense resolution, without importing Combat
- **TestDummyEnemy** now calls `NotifyDefenseSuccess()` on the target's `IDefenseProvider` after resolving defense outcomes, enabling the player to see visual cues when their defense succeeds against enemy attacks
- **HitboxManager** integrates clash immunity: checks `ClashTracker.IsClashImmune()` before applying damage, skips if target was recently clashed

### Files Added
- `Scripts/Shared/Components/ClashTracker.cs`
- `Scripts/Combat/Debug/DefenseDebugUI.cs`

### Files Modified
- `Scripts/Shared/Interfaces/IDefenseProvider.cs` (added `NotifyDefenseSuccess`)
- `Scripts/Combat/Defense/DefenseSystem.cs` (implemented `NotifyDefenseSuccess`)
- `Scripts/Combat/Hitbox/HitboxManager.cs` (clash immunity integration)
- `Scripts/World/TestDummyEnemy.cs` (calls `NotifyDefenseSuccess` after defense resolution)
- `Editor/Prefabs/PlayerPrefabCreator.cs` (wires ClashTracker to player prefab)
- `Editor/Prefabs/TestDummyPrefabCreator.cs` (wires ClashTracker to TestDummy prefab)
- `Editor/Prefabs/MovementTestSceneCreator.cs` (adds DefenseDebugUI to TestDummy in scene)

### Notes
- This is follow-up work on T016 (DefenseSystem), not a new task. T016 acceptance criteria were already met; this adds clash cancellation mechanics and debug visual feedback
- Work is uncommitted on `tal` branch

---

## [Phase 2] — 2026-03-04 (T016 DefenseSystem — DONE)

### Completed
- **T016: DefenseSystem — Deflect/Clash/Dodge** — branch `tal` (commit `9bc3f0e`)
  - `DefenseResolver` (plain C# class): Pure testable logic resolving defense outcomes — Hit, Deflected, Clashed, or Dodged — based on timing windows and directional context
  - `DefenseSystem` (MonoBehaviour): Event-driven window tracking via `CharacterMotor.Dashed` + `ComboController.AttackStarted`. Manages active defense windows per entity
  - `DefenseConfig` SO: Configurable per-entity timing — deflect 0–150ms, clash 20–80ms, dodge 50–300ms
  - `DefenseBonus` strategy pattern with 4 character-specific implementations:
    - **BrutorDefenseBonus**: No slideback on successful deflect
    - **SlasherDefenseBonus**: Crit chance boost on deflect
    - **MysticaDefenseBonus**: Mana restore on successful defense
    - **ViperDefenseBonus**: Damage reflect on dodge
  - `IDamageable.ResolveIncoming()`: Target-side defense resolution before damage application
  - `IDefenseProvider`: New cross-pillar interface so World (EnemyBase) references defense without Combat import
  - `HitboxManager`: Replaced temporary damage shim with full defense resolution pipeline
  - `DefenseContext`, `DefenseState`: Supporting data types for the defense pipeline
  - EnemyBase + TestDummyEnemy + PlayerDamageable updated to integrate defense resolution
  - Unblocks: T017 (Character Passives), T024 (Animation Polish)

### Design Decisions
- DD-1 (T016): DefenseResolver is a plain C# class — fully unit-testable without Unity runtime, matching ComboStateMachine/CharacterStatCalculator pattern
- DD-2 (T016): Strategy pattern for character bonuses — each character's unique defense reward is a separate class, avoiding a switch statement in DefenseSystem
- DD-3 (T016): IDefenseProvider in Shared/Interfaces — allows World pillar to query defense state without importing Combat

### Tests Added
- `DefenseResolverTests.cs` — 22 edit-mode NUnit tests covering all defense states (Hit, Deflected, Clashed, Dodged) and edge cases

### Files Added
- `Scripts/Combat/Defense/DefenseResolver.cs`
- `Scripts/Combat/Defense/DefenseSystem.cs`
- `Scripts/Combat/Defense/DefenseConfig.cs`
- `Scripts/Combat/Defense/DefenseContext.cs`
- `Scripts/Combat/Defense/DefenseState.cs`
- `Scripts/Combat/Defense/DefenseBonus.cs`
- `Scripts/Combat/Defense/Bonuses/BrutorDefenseBonus.cs`
- `Scripts/Combat/Defense/Bonuses/SlasherDefenseBonus.cs`
- `Scripts/Combat/Defense/Bonuses/MysticaDefenseBonus.cs`
- `Scripts/Combat/Defense/Bonuses/ViperDefenseBonus.cs`
- `Scripts/Shared/Interfaces/IDefenseProvider.cs`
- `Tests/EditMode/Combat/Defense/DefenseResolverTests.cs`

### Files Modified
- `Scripts/Combat/Hitbox/HitboxManager.cs` (replaced temp damage shim with defense resolution)
- `Scripts/Combat/PlayerDamageable.cs` (defense integration)
- `Scripts/Shared/Interfaces/IDamageable.cs` (added `ResolveIncoming()`)
- `Scripts/World/EnemyBase.cs` (defense integration)
- `Scripts/World/TestDummyEnemy.cs` (defense integration)
- `Editor/Prefabs/CharacterPrefabConfig.cs` (defense config field)
- `Editor/Prefabs/PlayerPrefabCreator.cs` (wires DefenseSystem)

---

## [Bug Fix] — 2026-03-04 (Attack hitbox alignment + knockback recovery)

### Problem
1. All player attack hitboxes (both Mystica and Slasher prefabs) were positioned at y=0.1, far below the hurtbox center at y=0.6 — attacks were barely overlapping with enemy hurtboxes.
2. Enemies hit by knockback slid infinitely because `Rigidbody2D` has `linearDamping=0` and `gravityScale=0`, and no recovery mechanism existed.

### Fixes

1. **Attack hitbox y-offset raised from 0.1 → 0.5** on all hitboxes to align with hurtbox center (y=0.6):
   - **Player.prefab (Mystica):** Hitbox_Burst, Hitbox_Bolt, Hitbox_BigBurst — colliders + debug visuals
   - **Slasher.prefab:** Hitbox_Slash, Hitbox_WideSlash, Hitbox_Lunge, Hitbox_Spin — colliders + debug visuals
   - **MysticaCharacterCreator.cs** + **SlasherCharacterCreator.cs**: Editor scripts updated to match, preventing re-generation from reverting the fix

2. **Knockback recovery after 0.5s** (`Scripts/World/EnemyBase.cs`):
   - `ApplyKnockback()` now starts a coroutine that zeros `Rb.linearVelocity` after 0.5 seconds
   - If hit again during knockback, the old timer is cancelled and a fresh 0.5s starts
   - No hit-stun state — enemy can still act while sliding (AI not implemented yet)

### Files Modified
- `unity/TomatoFighters/Assets/Prefabs/Player/Player.prefab` (6 collider/visual offsets)
- `unity/TomatoFighters/Assets/Prefabs/Player/Slasher.prefab` (8 collider/visual offsets)
- `unity/TomatoFighters/Assets/Scripts/World/EnemyBase.cs` (+field, +coroutine)
- `unity/TomatoFighters/Assets/Editor/Characters/MysticaCharacterCreator.cs` (3 offsets)
- `unity/TomatoFighters/Assets/Editor/Characters/SlasherCharacterCreator.cs` (4 offsets)

---

## [Tooling] — 2026-03-04 (Multi-Character Animation Pipeline)

### Changes
- **AnimationForgeMetadata.cs** (`Editor/Animation/`): Added `CharacterAnimConfig` struct + `Characters` registry (Mystica, Slasher). New `Load(string sourceFolder)` and `GetSheetPath(string spritesFolder, ...)` overloads for per-character metadata loading
- **SpriteSheetImporter.cs** (`Editor/Animation/`): Added `ImportSpriteSheets(string sourceFolder)` + per-character menu items (`TomatoFighters/Import Sprite Sheets/Mystica`, `TomatoFighters/Import Sprite Sheets/Slasher`)
- **AnimationBuilder.cs** (`Editor/Animation/`): Added `BuildAnimations(string sourceFolder, string outputFolder)` + per-character menu items. Parameterized `CreateClip` and `BuildController` for character-specific output paths
- **SlasherCharacterCreator.cs** (`Editor/Characters/`): Changed CONTROLLER_PATH to `Slasher_Controller`
- **MysticaCharacterCreator.cs** (`Editor/Characters/`): Changed CONTROLLER_PATH to `Mystica_Controller`, PREFAB_PATH from `Player.prefab` to `Mystica.prefab`
- **CharacterSelectTestSceneCreator.cs** (`Editor/Characters/`): Updated Mystica entry to reference `Mystica.prefab`
- **MovementTestSceneCreator.cs** (`Editor/Prefabs/`): Refactored to expose `CreateTestScene(prefabPath, scenePath, characterType)`. Added `AssetDatabase.ImportAsset` before loading prefab to fix stale Library cache
- **PlayerPrefabCreator.cs** (`Editor/Prefabs/`): Changed Animator controller from SerializedObject to direct assignment (`animator.runtimeAnimatorController`). Added forced reimport after save
- **SlasherMovementTestSceneCreator.cs** (NEW `Editor/Characters/`): Per-character wrapper for Slasher movement test scene
- **MysticaMovementTestSceneCreator.cs** (NEW `Editor/Characters/`): Per-character wrapper for Mystica movement test scene

### Bug Fixes
- **Animator controller null on scene instantiation**: Unity Library cache held stale prefab data after `PrefabUtility.SaveAsPrefabAsset`. Fixed by calling `AssetDatabase.ImportAsset(path, ImportAssetOptions.ForceUpdate)` after save and before load

### Notes
- Animation pipeline is now fully parameterized: Import Sprites → Build Animations → Create Character → Create Movement Scene, each with per-character menu items
- Work is uncommitted (in progress on `tal` branch)

---

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
