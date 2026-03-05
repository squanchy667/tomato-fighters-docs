# T013: Basic Test Scene

## Metadata
| Field | Value |
|-------|-------|
| **Phase** | 1 — Foundation |
| **Type** | implementation |
| **Priority** | P1 |
| **Owner** | Dev 3 |
| **Agent** | integration-agent |
| **Depends On** | T002, T010, T011, T012 |
| **Blocks** | — |
| **Status** | BLOCKED (waiting on T010, T012) |
| **Branch** | `pillar3/T013-test-scene` |

## Objective
Create a minimal test scene with flat ground, walls, a player spawn, dummy enemies, and camera setup — used for Phase 1 integration testing of controller movement, wave spawning, enemy damage, and camera follow.

## Context
This is the Phase 1 integration task. It wires together the foundational systems from all 3 pillars into a playable test scene. The scene validates that:
- `CharacterMotor` (T002, Combat pillar) moves and dashes correctly ✅
- `WaveManager` (T010, World pillar) spawns enemies — **NOT YET IMPLEMENTED**
- `EnemyBase` / `TestDummyEnemy` (T011, World pillar) takes damage and dies ✅
- `CameraController2D` (T012, World pillar) follows the player and respects bounds — **NOT YET IMPLEMENTED**

**Important**: Since CLI agents cannot open the Unity Editor, this task produces **Editor Creator Scripts** that programmatically create the scene, GameObjects, components, and wire serialized fields.

## Implementation Status

### Already Implemented (exceeds original spec)

The following Creator Scripts already deliver the core T013 functionality. Rather than creating a separate `TestArena.unity`, the existing `MovementTest.unity` scene serves as the integration test scene and will be evolved to add CameraController2D and WaveManager when those systems are built.

| Requirement | Status | Implemented By |
|---|---|---|
| Editor setup script | ✅ Done | `Editor/Prefabs/MovementTestSceneCreator.cs` |
| Menu accessible | ✅ Done | `TomatoFighters > Create Movement Test Scene` (+ per-character variants) |
| Flat ground + arena background | ✅ Done | 20×10 arena with grid lines for depth perception |
| 4 walls (L/R/Top/Bottom) | ✅ Done | Invisible BoxCollider2D walls |
| Player with full components | ✅ Done | Prefab instantiation: CharacterMotor, ComboController, CharacterInputHandler, DefenseSystem, PlayerDamageable, DebugHealthBar, DefenseDebugUI |
| Input wiring (WASD/Space/Shift/LMB/C/Ctrl) | ✅ Done | InputActionReference re-wiring on scene instance |
| Dummy enemies | ✅ **Exceeds spec** | 5 tiered dummies (Bruiser→Weakling) with unique AttackData, telegraph types, defense systems |
| DummyEnemy subclass | ✅ Done | `Scripts/World/TestDummyEnemy.cs` — AI with facing, telegraph, timed attacks |
| EnemyData SO | ✅ Done | Per-tier AttackData SOs created by `TestDummyPrefabCreator` |
| TestDummy prefab | ✅ Done | `Editor/Prefabs/TestDummyPrefabCreator.cs` — Rigidbody2D, DefenseSystem, ClashTracker, hitbox, debug health bar |
| Layer collision matrix | ✅ Done | PlayerHitbox↔EnemyHurtbox, EnemyHitbox↔PlayerHurtbox enabled; same-team disabled |
| Debug UI | ✅ Done | Health bars, defense event popups, combo debug overlay, controls hint |
| Character switching scene | ✅ **Bonus** | `Editor/Characters/CharacterSelectTestSceneCreator.cs` — press 1-4 to swap characters |
| Clash timing test scene | ✅ **Bonus** | `Editor/Prefabs/ClashTimingTestSceneCreator.cs` — automated clash timing validation |
| All 4 character prefabs | ✅ **Bonus** | Brutor, Slasher, Mystica, Viper creators with unique combos, hitboxes, defense configs |
| Animation pipeline | ✅ **Bonus** | `SpriteSheetImporter` + `AnimationBuilder` — metadata-driven sprite slicing and AnimatorController generation |

### Blocked — Waiting on T010, T012

| Requirement | Blocked On | What to Do When Unblocked |
|---|---|---|
| Camera with `CameraController2D` | T012 | Add `CameraController2D` to the camera in `MovementTestSceneCreator.SetupCamera()`. Wire follow target to player transform, configure orthographic bounds to arena width, wire SO event channels (stun zoom, bound lock). |
| `WaveManager` with test wave | T010 | Add a `WaveManager` GO in `MovementTestSceneCreator.CreateTestScene()` after `CreateTestDummies()`. Configure 1 wave with 3 enemy spawns. Wire SO event channel references. Keep the static tiered dummies as a separate debug option. |
| Level bound triggers | T012 (camera bounds) | Add trigger colliders inside L/R walls for camera bound-locking. Only meaningful once CameraController2D exists. |

### File Map (Actual)

| File | Purpose | Status |
|------|---------|--------|
| `Editor/Prefabs/MovementTestSceneCreator.cs` | Main test scene creator (replaces spec's `TestArenaSetup.cs`) | ✅ Done |
| `Editor/Prefabs/TestDummyPrefabCreator.cs` | TestDummy prefab + tiered AttackData factory | ✅ Done |
| `Editor/Prefabs/PlayerPrefabCreator.cs` | Generic player prefab builder (used by all character creators) | ✅ Done |
| `Editor/Prefabs/ClashTimingTestSceneCreator.cs` | Clash timing automated test scene | ✅ Done |
| `Editor/Characters/CharacterSelectTestSceneCreator.cs` | Character switching test scene | ✅ Done |
| `Editor/Characters/BrutorCharacterCreator.cs` | Brutor prefab creator | ✅ Done |
| `Editor/Characters/SlasherCharacterCreator.cs` | Slasher prefab creator | ✅ Done |
| `Editor/Characters/MysticaCharacterCreator.cs` | Mystica prefab creator | ✅ Done |
| `Editor/Characters/ViperCharacterCreator.cs` | Viper prefab creator | ✅ Done |
| `Scripts/World/TestDummyEnemy.cs` | Concrete enemy with AI, telegraph, attacks | ✅ Done |
| `Scenes/MovementTest.unity` | Output scene (generated, not hand-edited) | ✅ Done |

## Design Decisions

### DD-1: Evolve existing scene, don't create a separate TestArena
- **Decision:** Use `MovementTestSceneCreator` → `MovementTest.unity` as the integration test scene rather than creating a new `TestArenaSetup.cs` → `TestArena.unity`.
- **Rationale:** The existing scene already exceeds T013 requirements (5 tiered enemies, full defense system, debug UI, multiple character support). Creating a separate scene would duplicate 90% of the code. When T010/T012 are done, we add CameraController2D and WaveManager to the existing creator.

### DD-2: Block on T010/T012, don't stub
- **Decision:** Mark T013 as BLOCKED rather than stubbing CameraController2D/WaveManager integration points now.
- **Rationale:** The existing scene is fully playable for combat testing without camera follow or wave spawning. Adding stubs would just be dead code. When T010/T012 land, the integration points in `MovementTestSceneCreator` are straightforward: 3-5 lines in `SetupCamera()` and a new `CreateWaveManager()` method.

### DD-3: Keep static dummies alongside future WaveManager
- **Decision:** When WaveManager is added, keep the 5 tiered static dummies as a parallel debug option (possibly behind a bool flag in the creator).
- **Rationale:** Static dummies are invaluable for testing specific defense interactions (deflect vs Unstoppable Bruiser, clash timing vs Fighter). Wave-spawned enemies test flow; static dummies test mechanics.

## Acceptance Criteria

### Done
- [x] Editor Creator Script generates a complete test scene programmatically
- [x] Script accessible via Unity menu
- [x] Scene has: arena background, 4 wall colliders, grid lines
- [x] Player instantiated from prefab with: CharacterMotor, ComboController, CharacterInputHandler, DefenseSystem, PlayerDamageable, DebugHealthBar
- [x] Input actions wired via SerializedObject (Move, Jump, Dash, Light, Heavy, Run)
- [x] 5 tiered enemies with EnemyBase subclass, Rigidbody2D, BoxCollider2D, SpriteRenderer, DefenseSystem
- [x] Each enemy tier has unique AttackData SO with distinct damage/knockback/telegraph
- [x] Layer collision matrix configured (cross-team hits only)
- [x] Scene saved via `EditorSceneManager.SaveScene`
- [x] All serialized field references wired via `SerializedObject` (persist correctly)
- [x] Compiles with zero warnings

### Blocked (T010, T012)
- [ ] Camera has `CameraController2D` with follow target set to player
- [ ] Camera respects arena bounds, supports stun zoom event
- [ ] `WaveManager` present with at least 1 configured wave
- [ ] Level bound triggers inside L/R walls

## References
- [System Overview](../../architecture/system-overview.md) — Module architecture
- [Data Flow](../../architecture/data-flow.md) — Single combat frame flow (what this scene tests)
- [TASK_BOARD.md](../../TASK_BOARD.md) — T013 entry, dependencies on T002/T010/T011/T012
