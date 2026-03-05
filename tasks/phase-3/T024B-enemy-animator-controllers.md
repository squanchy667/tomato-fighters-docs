# T024B: Enemy Animator Controllers

| Field | Value |
|-------|-------|
| **Phase** | 3 — Defensive Depth + Build Crafting |
| **Type** | implementation |
| **Priority** | P1 |
| **Owner** | Dev 3 (World) |
| **Depends on** | T024 (Character Animator Controllers — DONE), T011 (EnemyBase — DONE) |
| **Blocks** | T022 (BasicEnemyAI), T023 (Enemy Attack Patterns), T042 (2nd Enemy Type + Mini-Bosses) |
| **Status** | DONE |
| **Branch** | `ofek` |
| **Completed** | 2026-03-05 |

## Summary

Extend the animation pipeline (AnimationForgeMetadata, AnimationBuilder, AnimationEventStamper) to support enemy types. Build a Base Enemy Animator Controller with a shared state machine and per-enemy-type Override Controllers for all 6 enemy types + TestDummy. Create a generic `EnemyPrefabCreator` builder (parallel to `PlayerPrefabCreator`) and update `TestDummyPrefabCreator` to wire an animator as proof of concept.

## Enemy Roster

| # | Display Name | Registry Key | Source Folder | Output Folder | character_name (metadata.json) |
|---|-------------|-------------|---------------|---------------|-------------------------------|
| 1 | Tomato Berserker | `TomatoBerserker` | `Assets/animations/tomato_berserker_animations` | `Assets/Animations/Enemies/TomatoBerserker` | `tomato_berserker` |
| 2 | Corn Knight | `CornKnight` | `Assets/animations/corn_knight_animations` | `Assets/Animations/Enemies/CornKnight` | `corn_knight` |
| 3 | Onion Crybaby Knight | `OnionCrybabyKnight` | `Assets/animations/onion_crybaby_knight_animations` | `Assets/Animations/Enemies/OnionCrybabyKnight` | `onion_crybaby_knight` |
| 4 | Garlic Vampire | `GarlicVampire` | `Assets/animations/garlic_vampire_animations` | `Assets/Animations/Enemies/GarlicVampire` | `garlic_vampire` |
| 5 | Eggplant Wizard | `EggplantWizard` | `Assets/animations/eggplant_wizard_animations` | `Assets/Animations/Enemies/EggplantWizard` | `eggplant_wizard` |
| 6 | Mushroom Ghost | `MushroomGhost` | `Assets/animations/mushroom_ghost_animations` | `Assets/Animations/Enemies/MushroomGhost` | `mushroom_ghost` |
| 7 | Test Dummy | `TestDummy` | *(no source folder — all placeholders)* | `Assets/Animations/Enemies/TestDummy` | *(none)* |

## Requirements

1. **Enemy Canonical State Registry** — Define enemy states in AnimationForgeMetadata, separate from player states: locomotion (idle, walk), attacks (attack_1–attack_5), reactions (hurt, death)
2. **Enemy Character Configs** — All 7 enemy entries (6 real + TestDummy) in `EnemyCharacters` registry with source/output folder mappings
3. **Base Enemy Controller** — `Assets/Animations/Enemies/Base/BaseEnemy_Controller.controller` with all enemy canonical states, using Corn Knight's clips as template (DD-5)
4. **Per-Enemy Override Controllers** — `Assets/Animations/Enemies/{Type}/{Type}_Override.overrideController` for all 7 enemies
5. **AnimationBuilder Extension** — Generate enemy base + override controllers alongside player controllers
6. **AnimationEventStamper Extension** — Stamp hitbox events on enemy attack clips from AttackData SOs
7. **Placeholder Clip Generation** — Single-frame placeholder clips for enemies without art or missing states. TestDummy uses WhiteSquare sprite; real enemies use their idle sprite
8. **Discovery-Driven Validation** — Clear ERROR for every animation name in metadata.json that doesn't map to an enemy canonical state, listing all unmapped names so devs can add mappings incrementally
9. **EnemyPrefabConfig** — Data class holding everything needed to build an enemy prefab (parallel to `CharacterPrefabConfig`)
10. **EnemyPrefabCreator** — Generic builder that per-enemy creator scripts delegate to (parallel to `PlayerPrefabCreator`)
11. **TestDummy Proof of Concept** — Update `TestDummyPrefabCreator` to wire animator + override controller
12. **EnemyBase.Die() Fix** — Change `GetComponent<Animator>()` → `GetComponentInChildren<Animator>()` (Animator lives on Sprite child)

## Design Decisions

### DD-1: Base + Override Pattern (Same as T024)

Reuse the same `AnimatorOverrideController` pattern from T024. One `BaseEnemy_Controller.controller` defines the shared state machine; each enemy type gets an override controller that swaps in its clips.

**Rationale:** Same benefits as player controllers — a transition change applies to all enemy types at once. Consistent behavior across enemy variants.

**Structure:**
- `Assets/Animations/Enemies/Base/BaseEnemy_Controller.controller` — shared state machine
- `Assets/Animations/Enemies/{Type}/{Type}_Override.overrideController` — per-enemy clips

### DD-2: Simpler State Machine Than Players

Enemies need fewer states than players:

| Category | States | Driver |
|----------|--------|--------|
| Locomotion | `idle`, `walk` | Speed float parameter (threshold 0.1) |
| Attacks | `attack_1` through `attack_5` | Trigger-driven (`attack_1Trigger`–`attack_5Trigger`) |
| Reactions | `hurt`, `death` | Trigger-driven (`Hurt`, `Death`) — one-shot, return to idle |

**Not included:** run, jump, land, dash, block, guard — enemies don't use player movement/defense mechanics. If a future boss needs unique states, it can get a custom controller.

**5 attack slots** provides headroom:
- TestDummy: 1 attack (DummyPunch)
- BasicEnemy (T022): 2–3 attacks
- Heavy (T042): 2–3 attacks
- MiniBoss (T042): 3–5 attacks
- Boss (T032): may need custom controller if >5 patterns

### DD-3: Enemy Trigger Names Diverge from Players

Enemy triggers use shorter names than player triggers for states where `EnemyBase.cs` already references them:

| State | Enemy Trigger | Player Trigger | Reason |
|-------|-------------|---------------|--------|
| `death` | `"Death"` | `"DeathTrigger"` | Matches existing `EnemyBase.Die()` code |
| `hurt` | `"Hurt"` | `"HurtTrigger"` | Consistent short naming with Death |
| `attack_N` | `"attack_NTrigger"` | `"attack_NTrigger"` | Same pattern (no existing code to match) |

**Defined in:** `AnimationBuilder.cs` as `ENEMY_TRIGGER_OVERRIDES` dictionary (parallel to player `TRIGGER_OVERRIDES`).

### DD-4: Animator on Sprite Child + EnemyBase Fix

Animator component goes on the `Sprite` child GameObject (same as player pattern — Animator and SpriteRenderer co-located). This requires fixing `EnemyBase.Die()`:

```csharp
// Before (line 279):
var animator = GetComponent<Animator>();
// After:
var animator = GetComponentInChildren<Animator>();
```

**Rationale:** Consistent with player prefab architecture. Animator binds to SpriteRenderer on the same GameObject via empty binding path `""`.

### DD-5: Base Controller Uses Corn Knight's Clips as Template

Corn Knight's clips populate the base enemy controller states. Override controllers for all other enemies (including TestDummy) replace them.

**Rationale:** Unity's `AnimatorOverrideController` requires real clips with actual frame data in each state. Corn Knight is the tallest/most upright enemy, likely to have the most complete standard animation set. Same pattern as T024 using Mystica for players.

**Fallback:** If Corn Knight's metadata.json is missing or incomplete, the builder logs an error and falls back to generating all-placeholder clips from the first available enemy's idle sprite.

### DD-6: Discovery-Driven Validation (Attack Slot Mappings Populated Incrementally)

`EnemyAttackSlotMappings` starts **empty** for all enemies. When the pipeline runs:

1. Load `metadata.json` for each enemy
2. Auto-match any animation named `idle`, `walk`, `attack_1`–`attack_5`, `hurt`, `death` → canonical states
3. For any animation name that doesn't match a canonical state → **clear ERROR** listing all unmapped names:

```
[AnimationForgeMetadata] ERROR — TomatoBerserker: 3 unmapped animations in metadata.json:
  - "headbutt" — not a canonical enemy state. Add to EnemyAttackSlotMappings or rename to attack_N
  - "ground_pound" — not a canonical enemy state
  - "rage_spin" — not a canonical enemy state
```

4. **WARNING** for canonical states with no matching animation → placeholder generated (same as T024)

**Rationale:** Lets devs run the pipeline immediately with raw Animation Forge output, see exactly what needs mapping, and add entries to `EnemyAttackSlotMappings` one at a time. No upfront knowledge of each enemy's attack set required.

### DD-7: EnemyPrefabCreator Generic Builder

Parallel to `PlayerPrefabCreator.cs`. Accepts an `EnemyPrefabConfig` and builds:
- Root GameObject with Rigidbody2D, body collider
- `Sprite` child with SpriteRenderer + Animator (DD-4)
- EnemyBase subclass component (wired via SerializedObject)
- Animator with override controller
- DefenseSystem + ClashTracker
- DebugHealthBar
- Hitbox children (from config definitions)

Per-enemy creator scripts (like `TestDummyPrefabCreator`) populate the config and delegate.

### DD-8: Graceful Handling of Missing Source Folders

For enemies with no source folder or no `metadata.json` (e.g., TestDummy):
- Skip clip building from sprite sheets
- Generate all-placeholder clips for every enemy canonical state
- Use WhiteSquare sprite (from `TestDummyPrefabCreator.GetOrCreateWhiteSquareSprite()`) as the placeholder image
- Still build a valid override controller (all states → placeholder clips)

This ensures every enemy in the registry gets a functional override controller regardless of art status.

## File Plan

### Modified Files

| # | File | Changes |
|---|------|---------|
| 1 | `Editor/Animation/AnimationForgeMetadata.cs` | Add `EnemyLocomotionStates`, `EnemyAttackSlots`, `EnemyReactionStates`, `EnemyCanonicalStates`. Add `EnemyCharacters` config dict (7 entries). Add `EnemyAttackSlotMappings` (starts empty). Add `ValidateEnemyMetadata()` with clear error listing of all unmapped animation names. |
| 2 | `Editor/Animation/AnimationBuilder.cs` | Add `ENEMY_BASE_OUTPUT_FOLDER`, `ENEMY_TRIGGER_OVERRIDES` dict. Add `BuildEnemyBaseController()` (simpler state machine — idle/walk + 5 attacks + hurt/death). Add `BuildEnemyOverrideController()`. Add `BuildEnemyClips()` with WhiteSquare fallback for missing sources. Extend `BuildAllAnimations()` to process enemies after players. Add `GenerateEnemyPlaceholderClips()`. |
| 3 | `Editor/Animation/AnimationEventStamper.cs` | Extend `StampAllEvents()` to iterate `EnemyCharacters` after players. Add `LoadEnemyAttackData()` using `Attacks/Enemy/` path. Enemy clips don't stamp `OnComboWindowOpen` or `OnFinisherEnd` (enemies don't use ComboController). |
| 4 | `Editor/Prefabs/TestDummyPrefabCreator.cs` | Wire Animator component on `Sprite` child. Load and assign `TestDummy_Override.overrideController`. Delegate prefab building to `EnemyPrefabCreator` where possible. |
| 5 | `Scripts/World/EnemyBase.cs` | Line 279: `GetComponent<Animator>()` → `GetComponentInChildren<Animator>()` |

### New Files

| # | File | Purpose |
|---|------|---------|
| 6 | `Editor/Prefabs/EnemyPrefabConfig.cs` | Data class for enemy prefab configuration. Fields: `prefabPath`, `enemyType` (string), `enemyDataAsset` (EnemyData), `attackDatas` (AttackData[]), `defenseConfig` (DefenseConfig), `animatorController` (RuntimeAnimatorController), `bodySize` (Vector2), `bodyOffset` (Vector2), `hitboxDefinitions` (HitboxDefinition[]), `spriteColor` (Color), `bodySprite` (Sprite). |
| 7 | `Editor/Prefabs/EnemyPrefabCreator.cs` | Generic enemy prefab builder. `public static GameObject CreateEnemyPrefab(EnemyPrefabConfig config)`. Builds: root GO (layer, Rigidbody2D, body collider, EnemyBase subclass via config), Sprite child (SpriteRenderer, Animator), DefenseSystem + ClashTracker, DebugHealthBar, hitbox children. |

### Generated Outputs (not hand-edited)

- `Assets/Animations/Enemies/Base/BaseEnemy_Controller.controller`
- `Assets/Animations/Enemies/{Type}/{Type}_Override.overrideController` × 7 (all enemies)
- `.anim` clips per enemy (real from sprite sheets or placeholder)

## Execution Order

1. **AnimationForgeMetadata.cs** — add enemy canonical state registry, 7 enemy character configs, empty attack slot mappings, `ValidateEnemyMetadata()` with clear unmapped animation error logging
2. **EnemyPrefabConfig.cs** — new data class for enemy prefab configuration
3. **EnemyPrefabCreator.cs** — new generic enemy prefab builder
4. **AnimationBuilder.cs** — extend to generate enemy base controller (Corn Knight template) + 7 override controllers, with WhiteSquare fallback for enemies without art
5. **AnimationEventStamper.cs** — extend to stamp events on enemy attack clips (ActivateHitbox/DeactivateHitbox only, no combo events)
6. **TestDummyPrefabCreator.cs** — wire Animator on Sprite child + override controller
7. **EnemyBase.cs** — fix `GetComponent<Animator>()` → `GetComponentInChildren<Animator>()`

## Pipeline Usage (after implementation)

```
# Full pipeline (processes both players AND enemies):
TomatoFighters > Import Sprite Sheets > All Characters    # Step 1: slice sprites
TomatoFighters > Build Animations > All Characters         # Step 2: player + enemy clips, controllers, overrides
TomatoFighters > Stamp Animation Events                    # Step 3: player + enemy hitbox events

# After adding a new enemy type:
1. Add entry to AnimationForgeMetadata.EnemyCharacters (+ EnemyAttackSlotMappings when ready)
2. Place metadata.json + sprite sheets in Assets/animations/{name}_animations/
3. Run the 3-step pipeline
4. Check console for unmapped animation ERRORs → add slot mappings as needed
5. Create {Type}EnemyCreator.cs that delegates to EnemyPrefabCreator
6. Re-run pipeline to stamp events after adding AttackData SOs

# Discovery workflow (new enemy with unknown attack names):
1. Place Animation Forge output in source folder
2. Run Build Animations
3. Read ERROR log → tells you exactly which animation names need mapping
4. Add entries to EnemyAttackSlotMappings
5. Re-run Build Animations + Stamp Events
```

## Risks & Gotchas

1. **TestDummy has no source art** — Pipeline must handle missing source folder gracefully by generating all-placeholder clips from WhiteSquare sprite. AnimationBuilder should not error on missing metadata.json for TestDummy — just log a warning and generate placeholders.
2. **TestDummyEnemy attack loop** — Currently drives attacks via timer + sprite color. After this task, attacks should trigger animator states. The attack loop in `TestDummyEnemy.cs` may need minor updates to set animator triggers (scope creep warning — keep changes minimal, wire trigger only).
3. **Corn Knight metadata may not be placed yet** — If Corn Knight's source folder is missing when pipeline runs, fall back to first available enemy's clips for the base controller template. If NO enemies have metadata, use all-placeholder WhiteSquare clips.
4. **Boss controllers** — Bosses (T032) with >5 attack patterns may need a custom extended controller. This task should document how to extend but not implement it.
5. **Enemy stamper skips combo events** — Unlike players, enemies don't use ComboController. Only stamp `ActivateHitbox` and `DeactivateHitbox` on enemy attack clips. Do NOT stamp `OnComboWindowOpen` or `OnFinisherEnd`.
6. **SpriteSheetImporter (Step 1) extension** — The existing `Import Sprite Sheets > All Characters` menu item only iterates `AnimationForgeMetadata.Characters` (players). It needs extending to also iterate `EnemyCharacters` to slice enemy sprite sheets. Add this alongside the AnimationBuilder extension.

## Acceptance Criteria

- [x] Base enemy controller with states: idle, walk, attack_1–attack_5, hurt, death
- [x] 7 override controllers generated (TomatoBerserker, CornKnight, OnionCrybabyKnight, GarlicVampire, EggplantWizard, MushroomGhost, TestDummy)
- [x] TestDummy prefab wired with Animator on Sprite child + override controller
- [x] AnimationForgeMetadata extended with enemy canonical states, 7 character configs, and attack slot mappings
- [x] AnimationBuilder generates enemy base + override controllers (menu: Build Animations > All Characters)
- [x] AnimationEventStamper stamps ActivateHitbox/DeactivateHitbox on enemy attack clips
- [x] Placeholder clips generated for enemies without animation art (WhiteSquare fallback)
- [x] Clear ERROR logged listing all unmapped animation names per enemy (discovery workflow)
- [x] WARNING logged for canonical states with no matching animation
- [x] EnemyPrefabConfig + EnemyPrefabCreator exist, TestDummyPrefabCreator delegates to it
- [x] EnemyBase.Die() uses `GetComponentInChildren<Animator>()`
- [x] SpriteSheetImporter extended to slice enemy sprite sheets
- [x] Pipeline runs end-to-end without errors when enemy source folders exist with valid metadata

## References

- T024 spec: `tasks/phase-2/T024-character-animator-controllers.md` — player animator pattern (direct parallel)
- T011 spec: `tasks/phase-1/T011-enemy-base.md` — EnemyBase architecture
- `Editor/Prefabs/TestDummyPrefabCreator.cs` — current enemy prefab creator (to be updated)
- `Editor/Prefabs/PlayerPrefabCreator.cs` — player prefab builder pattern to follow
- `Editor/Animation/AnimationForgeMetadata.cs` — central registry to extend
- `Editor/Animation/AnimationBuilder.cs` — animation pipeline Step 2
- `Editor/Animation/AnimationEventStamper.cs` — animation pipeline Step 3
- `developer/unity-editor-scripts.md` — Creator Script reference
- Enemy visual briefing — 6 enemies: Tomato Berserker, Corn Knight, Onion Crybaby Knight, Garlic Vampire, Eggplant Wizard, Mushroom Ghost
