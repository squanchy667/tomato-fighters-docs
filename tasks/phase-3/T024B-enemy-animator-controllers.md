# T024B: Enemy Animator Controllers

| Field | Value |
|-------|-------|
| **Phase** | 3 — Defensive Depth + Build Crafting |
| **Type** | implementation |
| **Priority** | P1 |
| **Owner** | Dev 3 (World) |
| **Depends on** | T024 (Character Animator Controllers — DONE), T011 (EnemyBase — DONE) |
| **Blocks** | T022 (BasicEnemyAI), T023 (Enemy Attack Patterns), T042 (2nd Enemy Type + Mini-Bosses) |
| **Status** | PENDING |
| **Branch** | `world/T024B-enemy-animators` |

## Summary

Extend the animation pipeline (AnimationForgeMetadata, AnimationBuilder, AnimationEventStamper) to support enemy types. Build a Base Enemy Animator Controller with a shared state machine and per-enemy-type Override Controllers. Create a generic `EnemyPrefabCreator` builder (parallel to `PlayerPrefabCreator`) and update `TestDummyPrefabCreator` to wire an animator as proof of concept.

## Requirements

1. **Enemy Canonical State Registry** — Define enemy states in AnimationForgeMetadata, separate from player states: locomotion (idle, walk), attacks (attack_1–attack_5), reactions (hurt, death)
2. **Enemy Character Configs** — Add enemy entries to AnimationForgeMetadata with source/output folder mappings and attack slot mappings
3. **Base Enemy Controller** — `Assets/Animations/Enemies/Base/BaseEnemy_Controller.controller` with all enemy canonical states
4. **Per-Enemy Override Controllers** — `Assets/Animations/Enemies/{Type}/{Type}_Override.overrideController` per enemy type
5. **AnimationBuilder Extension** — Generate enemy base + override controllers alongside player controllers
6. **AnimationEventStamper Extension** — Stamp hitbox/combo events on enemy attack clips from AttackData SOs
7. **Placeholder Clip Generation** — Single-frame placeholder clips for enemies without art (same pattern as T024)
8. **EnemyPrefabConfig** — Data class holding everything needed to build an enemy prefab (parallel to `CharacterPrefabConfig`)
9. **EnemyPrefabCreator** — Generic builder that per-enemy creator scripts delegate to (parallel to `PlayerPrefabCreator`)
10. **TestDummy Proof of Concept** — Update `TestDummyPrefabCreator` to wire animator + override controller

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
| Locomotion | `idle`, `walk` | Speed float parameter |
| Attacks | `attack_1` through `attack_5` | Trigger-driven (`attack_1Trigger`–`attack_5Trigger`) |
| Reactions | `hurt`, `death` | Trigger-driven (one-shot, return to idle) |

**Not included:** run, jump, land, dash, block, guard — enemies don't use player movement/defense mechanics. If a future boss needs unique states, it can get a custom controller.

**5 attack slots** provides headroom:
- TestDummy: 1 attack (DummyPunch)
- BasicEnemy (T022): 2–3 attacks
- Heavy (T042): 2–3 attacks
- MiniBoss (T042): 3–5 attacks
- Boss (T032): may need custom controller if >5 patterns

### DD-3: Enemy Configs in AnimationForgeMetadata

Add parallel data structures for enemies:
- `EnemyCharacters` dictionary (source folder → output folder mapping)
- `EnemyAttackSlotMappings` dictionary (slot → AttackData SO name)
- `EnemyCanonicalStates` set (union of enemy locomotion + attacks + reactions)

Kept separate from player registries so `BuildAllAnimations()` can process players and enemies independently.

### DD-4: EnemyPrefabCreator Generic Builder

Parallel to `PlayerPrefabCreator.cs`. Accepts an `EnemyPrefabConfig` and builds:
- Root GameObject with Rigidbody2D, body collider, SpriteRenderer
- EnemyBase subclass component (wired via SerializedObject)
- Animator with override controller
- DefenseSystem + ClashTracker
- DebugHealthBar
- Hitbox children (from config definitions)

Per-enemy creator scripts (like `TestDummyPrefabCreator`) populate the config and delegate.

### DD-5: Base Controller Uses TestDummy Clips as Template

Same rationale as T024 DD-5 — Unity's `AnimatorOverrideController` requires real clips with frame data. TestDummy is the first enemy type built, so its clips (or placeholders from its idle sprite) populate the base controller.

### DD-6: Validation Rules (Same as T024)

- **ERROR** if enemy metadata has animations that don't map to any enemy canonical state
- **WARNING** if an enemy canonical state has no matching animation (placeholder generated)

## File Plan

### Modified Files

| # | File | Changes |
|---|------|---------|
| 1 | `Editor/Animation/AnimationForgeMetadata.cs` | Add `EnemyCanonicalStates` registry, `EnemyCharacters` config dict, `EnemyAttackSlotMappings`. Add `ValidateEnemyMetadata()`. |
| 2 | `Editor/Animation/AnimationBuilder.cs` | Add `BuildEnemyBaseController()`, `BuildEnemyOverrideController()`. Extend `BuildAllAnimations()` to process enemies. |
| 3 | `Editor/Animation/AnimationEventStamper.cs` | Extend `StampAllEvents()` to iterate enemy characters and stamp events from enemy AttackData SOs. |
| 4 | `Editor/Prefabs/TestDummyPrefabCreator.cs` | Wire Animator component + override controller on TestDummy prefab. |

### New Files

| # | File | Purpose |
|---|------|---------|
| 5 | `Editor/Prefabs/EnemyPrefabConfig.cs` | Data class for enemy prefab configuration (parallel to `CharacterPrefabConfig`). Fields: prefabPath, enemyData, attackDatas, defenseConfig, animatorController, hitboxes. |
| 6 | `Editor/Prefabs/EnemyPrefabCreator.cs` | Generic enemy prefab builder. Accepts `EnemyPrefabConfig`, creates root GO with all components, wires references. Per-enemy creator scripts delegate to this. |

### Generated Outputs (not hand-edited)

- `Assets/Animations/Enemies/Base/BaseEnemy_Controller.controller`
- `Assets/Animations/Enemies/TestDummy/TestDummy_Override.overrideController`
- Placeholder `.anim` clips for TestDummy

## Execution Order

1. **AnimationForgeMetadata.cs** — add enemy canonical state registry, enemy character configs, enemy attack slot mappings
2. **EnemyPrefabConfig.cs** — new data class for enemy prefab configuration
3. **EnemyPrefabCreator.cs** — new generic enemy prefab builder
4. **AnimationBuilder.cs** — extend to generate enemy base controller + override controllers
5. **AnimationEventStamper.cs** — extend to stamp events on enemy attack clips
6. **TestDummyPrefabCreator.cs** — wire animator + override controller as proof of concept

## Pipeline Usage (after implementation)

```
# Full pipeline (processes both players AND enemies):
TomatoFighters > Import Sprite Sheets > All Characters    # Step 1: slice sprites
TomatoFighters > Build Animations > All Characters         # Step 2: player + enemy clips, controllers, overrides
TomatoFighters > Stamp Animation Events                    # Step 3: player + enemy hitbox/combo events

# After adding a new enemy type:
1. Add entry to AnimationForgeMetadata.EnemyCharacters + EnemyAttackSlotMappings
2. Create/place metadata.json + sprite sheets in source folder
3. Create {Type}EnemyCreator.cs (editor script) that delegates to EnemyPrefabCreator
4. Re-run the 3-step pipeline
```

## Risks & Gotchas

1. **No enemy animation art yet** — TestDummy currently uses sprite color changes, not animations. All clips will be placeholders initially. The pipeline must work entirely with placeholders.
2. **TestDummyEnemy attack loop** — Currently drives attacks via timer + sprite color. After this task, attacks should trigger animator states. The attack loop in `TestDummyEnemy.cs` may need minor updates to set animator triggers (scope creep warning — keep changes minimal, wire trigger only).
3. **EnemyBase.OnDeath already sets Death trigger** — `EnemyBase.cs` line ~280 already has `animator.SetTrigger("Death")`. Ensure the base enemy controller's death state trigger name matches.
4. **Boss controllers** — Bosses (T032) with >5 attack patterns may need a custom extended controller. This task should document how to extend but not implement it.
5. **Enemy metadata.json may not exist** — AnimationBuilder should gracefully handle enemies with no source folder/metadata (generate all-placeholder override controllers from idle sprite).

## Acceptance Criteria

- [ ] Base enemy controller with states: idle, walk, attack_1–attack_5, hurt, death
- [ ] Override controller infrastructure (per-enemy-type, TestDummy as first)
- [ ] TestDummy prefab wired with Animator + override controller
- [ ] AnimationForgeMetadata extended with enemy canonical states, character configs, and attack slot mappings
- [ ] AnimationBuilder generates enemy base + override controllers (menu: Build Animations > All Characters)
- [ ] AnimationEventStamper stamps events on enemy attack clips (menu: Stamp Animation Events)
- [ ] Placeholder clips generated for enemies without animation art
- [ ] ERROR/WARNING validation for enemy metadata (same rules as T024)
- [ ] EnemyPrefabCreator generic builder exists and TestDummyPrefabCreator delegates to it
- [ ] New enemy types can be added by creating a creator script + metadata entry (documented pattern)

## References

- T024 spec: `tasks/phase-2/T024-character-animator-controllers.md` — player animator pattern (direct parallel)
- T011 spec: `tasks/phase-1/T011-enemy-base.md` — EnemyBase architecture
- `Editor/Prefabs/TestDummyPrefabCreator.cs` — current enemy prefab creator (to be updated)
- `Editor/Prefabs/PlayerPrefabCreator.cs` — player prefab builder pattern to follow
- `Editor/Animation/AnimationForgeMetadata.cs` — central registry to extend
- `developer/unity-editor-scripts.md` — Creator Script reference
