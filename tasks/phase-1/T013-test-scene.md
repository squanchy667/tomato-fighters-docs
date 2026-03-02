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
| **Status** | PENDING |
| **Branch** | `pillar3/T013-test-scene` |

## Objective
Create a minimal test scene with flat ground, walls, a player spawn, dummy enemies, and camera setup — used for Phase 1 integration testing of controller movement, wave spawning, enemy damage, and camera follow.

## Context
This is the Phase 1 integration task. It wires together the foundational systems from all 3 pillars into a playable test scene at `Assets/Scenes/TestArena.unity`. The scene validates that:
- `CharacterController2D` (T002, Combat pillar) moves and jumps correctly
- `WaveManager` (T010, World pillar) spawns enemies
- `EnemyBase` (T011, World pillar) takes damage and dies
- `CameraController2D` (T012, World pillar) follows the player and respects bounds

**Important**: Since CLI agents cannot open the Unity Editor, this task must produce an **Editor setup script** that programmatically creates the scene, GameObjects, components, and wires serialized fields. This follows the KingOfOpera pattern (`Editor/ProjectSetup.cs`). The setup script runs from Unity's Tools menu.

## Requirements
1. Create an Editor setup script at `Assets/Scripts/Editor/TestArenaSetup.cs`
2. The script must be accessible via Unity menu: `Tools > TomatoFighters > Setup Test Arena`
3. The setup script programmatically creates a new scene containing:

   **Ground:**
   4. Flat ground plane — `GameObject` with `BoxCollider2D` (wide, thin), `SpriteRenderer` (placeholder color), positioned at Y=0
   5. Ground should span enough width for a combat arena (e.g., 30-40 units wide)

   **Walls:**
   6. Left wall — `GameObject` with `BoxCollider2D` (tall, thin), positioned at left edge of ground
   7. Right wall — `GameObject` with `BoxCollider2D` (tall, thin), positioned at right edge of ground
   8. Walls should be tall enough to prevent jumping over (e.g., 10 units)

   **Level Bounds:**
   9. Left level bound trigger — `BoxCollider2D` set as trigger, positioned slightly inside left wall
   10. Right level bound trigger — `BoxCollider2D` set as trigger, positioned slightly inside right wall

   **Player:**
   11. Player spawn point — empty `GameObject` at a start position (e.g., X=-10, Y=1)
   12. Player `GameObject` with: `SpriteRenderer` (placeholder square), `Rigidbody2D` (gravity scale configured), `BoxCollider2D`, `CharacterController2D` component
   13. Wire `CharacterController2D` serialized fields with default values

   **Enemies:**
   14. 2-3 dummy enemy `GameObjects` with: `SpriteRenderer` (different placeholder color), `Rigidbody2D`, `BoxCollider2D`, `EnemyBase`-derived component (or a `DummyEnemy` MonoBehaviour that inherits `EnemyBase`)
   15. Create a basic `EnemyData` ScriptableObject asset with test values (100 HP, 50 pressure threshold, low knockback resistance)
   16. Wire enemy `[SerializeField] EnemyData` references

   **Camera:**
   17. Main Camera with `CameraController2D` component
   18. Set camera follow target to the player transform
   19. Configure orthographic size, bounds matching the arena width
   20. Wire SO event channel references (stun zoom, bound lock) if available

   **Wave Manager:**
   21. Empty `GameObject` with `WaveManager` component
   22. Configure 1 test wave with 2-3 enemy spawns
   23. Wire SO event channel references

4. The setup script should use `SerializedObject` and `SerializedProperty` to properly wire serialized field references (not just direct assignment, which doesn't persist in Unity)
5. Save the scene to `Assets/Scenes/TestArena.unity` using `EditorSceneManager.SaveScene`
6. Optionally create a `DummyEnemy` class at `Assets/Scripts/World/DummyEnemy.cs` that extends `EnemyBase` with minimal concrete implementations (just the abstract methods)

## File Plan
| File | Purpose |
|------|---------|
| `Assets/Scripts/Editor/TestArenaSetup.cs` | Editor script that programmatically creates the TestArena scene (Tools > TomatoFighters > Setup Test Arena) |
| `Assets/Scripts/World/DummyEnemy.cs` | Minimal concrete EnemyBase subclass for testing |
| `Assets/Scenes/TestArena.unity` | Output scene (created by the setup script, not hand-authored) |
| `Assets/ScriptableObjects/Enemies/DummyEnemyData.asset` | Test EnemyData SO (created by setup script) |

## Implementation Notes
- **Editor script pattern**: Use `[MenuItem("Tools/TomatoFighters/Setup Test Arena")]` on a static method. The script must live in an `Editor/` folder (or have `Editor` as assembly definition)
- **SerializedObject for wiring**: When setting `[SerializeField]` references, use:
  ```csharp
  var so = new SerializedObject(component);
  so.FindProperty("fieldName").objectReferenceValue = targetObject;
  so.ApplyModifiedProperties();
  ```
  Direct field assignment via reflection does not reliably persist in Unity's serialization
- **DummyEnemy**: Minimal subclass — override abstract methods with empty bodies or simple log statements. Set health from EnemyData. This is a test-only class
- **Placeholder sprites**: Use `Sprite.Create()` with a small texture, or reference Unity's built-in white square sprite (`Sprite` from `UnityEditor.EditorGUIUtility`)
- **Layer setup**: Consider setting player to "Player" layer and enemies to "Enemy" layer with appropriate collision matrix, but this can be deferred
- **Scene hierarchy organization**: Group objects under parent empties: `--- Environment ---`, `--- Player ---`, `--- Enemies ---`, `--- Systems ---`
- **No hardcoded absolute paths** in the setup script — use `AssetDatabase.CreateAsset` with relative project paths
- **Idempotent**: The setup script should handle being run multiple times (check if scene/assets exist, overwrite or skip)

## Acceptance Criteria
- [ ] Editor setup script exists at `Assets/Scripts/Editor/TestArenaSetup.cs`
- [ ] Script accessible via menu: `Tools > TomatoFighters > Setup Test Arena`
- [ ] Running the script creates a scene with: flat ground with collision, walls on both sides, player spawn point
- [ ] Player has `CharacterController2D`, `Rigidbody2D`, `BoxCollider2D`, `SpriteRenderer`
- [ ] 2-3 dummy enemies with `EnemyBase` subclass, `Rigidbody2D`, `BoxCollider2D`, `SpriteRenderer`
- [ ] Dummy enemies reference a test `EnemyData` ScriptableObject with basic stats
- [ ] Camera has `CameraController2D` with follow target set to player
- [ ] `WaveManager` present with at least 1 configured wave
- [ ] All serialized field references are wired via `SerializedObject` (persist after script runs)
- [ ] Scene saved to `Assets/Scenes/TestArena.unity`
- [ ] `DummyEnemy` compiles and extends `EnemyBase` correctly
- [ ] Compiles with zero warnings (in both runtime and editor assemblies)

## References
- [System Overview](../../architecture/system-overview.md) — Module architecture, all file paths
- [Data Flow](../../architecture/data-flow.md) — Single combat frame flow (what this scene tests)
- [TASK_BOARD.md](../../TASK_BOARD.md) — T013 entry, dependencies on T002/T010/T011/T012
- [development-agents.md](../../development-agents.md) — Batch 1.2c: T013 integration scene
- KingOfOpera precedent: `Editor/ProjectSetup.cs` pattern for programmatic scene creation
