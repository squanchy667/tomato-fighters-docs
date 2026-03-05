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
| **Status** | IN_PROGRESS (art layer upgrade; T010/T012 items still blocked) |
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

### DD-4: Upgrade existing CreateArenaBackground() with art layers
- **Decision:** Replace the programmatic colored-rectangle background in `MovementTestSceneCreator.CreateArenaBackground()` with the 6 forest environment sprites from `Assets/Art/Environment/TestArena/`.
- **Rationale:** The sprites are ready, the scene creator already exists and works — simplest path is to swap the background method rather than creating a separate TestArenaCreator. Keeps one scene creator, one scene.

### DD-5: Full-width stacked background layers
- **Decision:** All 3 background layers (`bg_forest_distant`, `bg_forest_midground`, `bg_forest_foreground`) span the full 20×10 arena dimensions, stacked via sorting order.
- **Rationale:** Simple and works for any sprite aspect ratio. True parallax scrolling (camera-relative movement) is a future concern when CameraController2D (T012) lands.

### DD-6: Stone wall sprites are visual-only, physics stays on invisible colliders
- **Decision:** Keep the existing invisible `BoxCollider2D` walls for physics. Overlay `wall_left_stone` and `wall_right_stone` as purely visual `SpriteRenderer` GameObjects with no colliders.
- **Rationale:** Decouples visual art from physics boundaries. The invisible colliders are proven to work; adding colliders to art sprites would create confusing double-collision scenarios.

### DD-7: Sprite scaling from native texture size
- **Decision:** Compute each sprite's scale by dividing the target arena dimension by the sprite's native world-space size (`sprite.bounds.size`). This auto-adapts to any PPU setting.
- **Rationale:** Avoids hardcoding pixel sizes or assuming a specific PPU. If the artist re-exports at different resolution, the scene creator self-corrects.
- **Code pattern:**
```csharp
var sprite = AssetDatabase.LoadAssetAtPath<Sprite>(path);
float scaleX = ARENA_WIDTH / sprite.bounds.size.x;
float scaleY = ARENA_HEIGHT / sprite.bounds.size.y;
go.transform.localScale = new Vector3(scaleX, scaleY, 1f);
```

### DD-8: Monster animations deferred
- **Decision:** Monster animations exist but are not yet imported into Unity. This task does NOT touch monster sprites. When they are imported, the `TestDummyPrefabCreator` or a new `MonsterAnimationBuilder` will handle them separately.
- **Rationale:** Art layers and monster animations are independent workstreams. Combining them would bloat the scope.

## Remaining Work (Art Layer Upgrade)

### Changes to `Editor/Prefabs/MovementTestSceneCreator.cs`

**Method: `CreateArenaBackground()`** — Rewrite to:

1. Create an `Environment` parent GameObject
2. Load 6 sprites from `Assets/Art/Environment/TestArena/`
3. Create child GameObjects with `SpriteRenderer` for each layer:

| # | Sprite File | GO Name | sortingOrder | Position | Notes |
|---|-------------|---------|-------------|----------|-------|
| 1 | `bg_forest_distant.png` | `BG_Distant` | -100 | (0, 0, 0) | Full arena, furthest back |
| 2 | `bg_forest_midground.png` | `BG_Midground` | -90 | (0, 0, 0) | Full arena, trees/foliage |
| 3 | `bg_forest_foreground.png` | `BG_Foreground` | -80 | (0, 0, 0) | Full arena, closest foliage |
| 4 | `ground_forest_floor.png` | `Ground_Floor` | -50 | (0, -ARENA_HEIGHT/2 + floorHeight/2, 0) | Bottom of arena, floor surface |
| 5 | `wall_left_stone.png` | `Wall_Left_Visual` | -40 | (-ARENA_WIDTH/2 + wallWidth/2, 0, 0) | Left edge, visual only |
| 6 | `wall_right_stone.png` | `Wall_Right_Visual` | -40 | (ARENA_WIDTH/2 - wallWidth/2, 0, 0) | Right edge, visual only |

4. Scale each sprite to fit using `sprite.bounds.size` (DD-7)
5. Remove the old `CreateRectSprite()`-based ground and grid lines

**Method: `CreateRectSprite()`** — Keep as-is (still used by `BuildInlineFallbackPlayer()`)

**No other methods change.** Walls, player, dummies, input wiring, debug UI — all untouched.

### Execution Order

1. Modify `CreateArenaBackground()` in `MovementTestSceneCreator.cs`
2. Re-run `TomatoFighters > Create Movement Test Scene` in Unity to regenerate the scene
3. Verify sprites load and display correctly in the Scene view

## Acceptance Criteria

### Done (Core Gameplay)
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

### Pending (Art Layer Upgrade)
- [ ] `CreateArenaBackground()` loads 6 sprites from `Assets/Art/Environment/TestArena/`
- [ ] 3 background layers (`bg_forest_distant`, `bg_forest_midground`, `bg_forest_foreground`) render full-arena stacked by sorting order
- [ ] `ground_forest_floor` renders at bottom of arena
- [ ] `wall_left_stone` and `wall_right_stone` render at arena edges (visual only, no colliders)
- [ ] All sprites scaled to fit arena via `sprite.bounds.size` calculation
- [ ] Old programmatic grid-line background removed
- [ ] Scene regenerates cleanly via menu with new art

### Blocked (T010, T012)
- [ ] Camera has `CameraController2D` with follow target set to player
- [ ] Camera respects arena bounds, supports stun zoom event
- [ ] `WaveManager` present with at least 1 configured wave
- [ ] Level bound triggers inside L/R walls

## References
- [System Overview](../../architecture/system-overview.md) — Module architecture
- [Data Flow](../../architecture/data-flow.md) — Single combat frame flow (what this scene tests)
- [TASK_BOARD.md](../../TASK_BOARD.md) — T013 entry, dependencies on T002/T010/T011/T012
