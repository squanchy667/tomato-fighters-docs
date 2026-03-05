# T024: Character Animator Controllers

**Phase:** 2 — Core Combat + Path Framework
**Priority:** P1
**Owner:** Dev 3 (World)
**Depends on:** T014 (ComboSystem — All 4 Characters)
**Blocks:** T025 (HUD), T034 (Path T1 Ability VFX)

## Summary

Build a Base Animator Controller with shared state machine and 4 Animator Override Controllers with character-specific clips. Wire Animation Events for hitbox activation, combo window, and finisher completion. Add placeholder clip generation for missing attack art with validation (error on unmapped metadata animations, warning on generated placeholders).

## Acceptance Criteria

- [ ] Base controller with all combat states (locomotion, airborne, dash, 10 attack slots, defense, hurt, death)
- [ ] 4 override controllers (Brutor, Slasher, Mystica, Viper)
- [ ] Animation Events stamped on attack clips for hitbox, combo window, finisher timing
- [ ] Transition conditions match combat system states
- [ ] ERROR logged for metadata animations that don't map to any canonical state
- [ ] WARNING logged for canonical states with no matching animation (placeholder generated)

## Design Decisions

### DD-1: Base + Override Architecture
Use Unity's AnimatorOverrideController pattern instead of per-character standalone controllers.

**Rationale:** All 4 characters share the same state machine (same states, same transitions, same parameters). Only the animation clips differ. Base + Override guarantees consistent behavior — a transition change applies to all characters at once. Prevents silent combo breakage on individual characters.

**Structure:**
- `Assets/Animations/Base/BaseCharacter_Controller.controller` — single source of truth for state machine
- `Assets/Animations/{Name}/{Name}_Override.overrideController` × 4 — character-specific clip mappings

### DD-2: Placeholder Clips for Missing Art
When a canonical state has no matching animation in a character's metadata.json, auto-generate a single-frame placeholder clip using the idle sprite sheet.

**Validation rules:**
- **ERROR** if metadata has animations that don't map to any canonical state → likely a naming mismatch or missed attack mapping
- **WARNING** if a canonical state has no matching animation → placeholder generated, art needed later

**Rationale:** Controllers are fully functional immediately. ComboController triggers work (character flashes to idle — obvious placeholder). No manual asset creation. Art can be swapped in later by updating metadata.json and re-running the pipeline.

### DD-3: Generic Attack Slots (10 slots)
Base controller has `attack_1` through `attack_10` as trigger-driven action states. No semantic naming (no `strike`, `heavy`, `finisher` distinction at the controller level).

**Rationale:** Combo trees vary per character (Brutor: 7 steps, Slasher: 8, Mystica: 5, Viper: 6). Each combo branch can end in a different finisher, requiring its own animation. The ComboDefinition maps each step to a trigger name (`attack_1Trigger`, `attack_2Trigger`, etc.). Whether a step is a finisher is determined by `ComboStep.isFinisher`, not by the animation state name.

**Slot mapping example:**
| Slot | Brutor | Slasher | Mystica | Viper |
|------|--------|---------|---------|-------|
| attack_1 | L1 Jab | L1 Slash | Strike1 | Shot1 |
| attack_2 | L2 Cross | L2 Slash2 | Strike2 | Shot2 |
| attack_3 | L3 Sweep (F) | L3 Slash3 | Strike3 (F) | RapidBurst (F) |
| attack_4 | H Launcher | Spin (F) | ArcaneBolt | Charged |
| attack_5 | AirSlam (F) | H1 Heavy | EmpoweredBolt (F) | Piercing (F) |
| attack_6 | — | Lunge | — | QuickCharged |
| attack_7 | — | PiercingLunge (F) | — | — |
| attack_8 | — | QuickSlash | — | — |

10 slots provides headroom for future path abilities.

### DD-6: Defense & Reaction States
Base controller includes defense and reaction states beyond attacks:

| State | Type | Trigger | Purpose |
|-------|------|---------|---------|
| `block` | one-shot | `blockTrigger` | Plays on deflect or dodge |
| `guard` | looping | `guardTrigger` | Sustained pose during clash window |
| `hurt` | one-shot | `HurtTrigger` | Hit reaction |
| `death` | one-shot | `DeathTrigger` | Death animation |

**Rationale:** Defense system (T016) needs visual feedback for deflect, dodge, and clash. `block` is a single animation covering both deflect and dodge. `guard` loops while the clash window is active. `hurt` matches the existing `HURTTRIGGER` constant in `TomatoFighterAnimatorParams.cs`.

**Viper sprite validation (reference for naming convention):**
- `yellow_block.png` → maps to `block` canonical state
- `yellow_guard.png` → maps to `guard` canonical state
- `yellow_hurt.png` → maps to `hurt` canonical state
- `yellow_death.png` → maps to `death` canonical state
- `yellow_attack_1.png` → maps to `attack_1`
- `yellow_attack_2.png` → maps to `attack_2`
- `yellow_kneel.png` → unmapped (deferred, future downed/stagger state)
- `yellow_fall.png` → unmapped (deferred, future extended airborne state)

### DD-4: AnimationEventStamper as Separate Pipeline Step 3
Animation Events are baked onto .anim clips at build time via a new `AnimationEventStamper` editor script, separate from AnimationBuilder.

**Rationale:** Keeps AnimationBuilder clean (clips + controller only). Events depend on AttackData SOs (hitboxStartFrame, hitboxActiveFrames, totalFrames), which are a different data source than metadata.json. Re-stamping events when timing changes doesn't require rebuilding all clips.

**Events stamped per attack clip:**
1. `ActivateHitbox()` at frame `hitboxStartFrame` — calls HitboxManager
2. `DeactivateActiveHitbox()` at frame `hitboxStartFrame + hitboxActiveFrames` — calls HitboxManager
3. `OnComboWindowOpen()` after hitbox ends — calls ComboController, opens input window
4. `OnFinisherEnd()` at last frame of finisher clips — calls ComboController

**Menu:** `TomatoFighters > Stamp Animation Events`

### DD-5: Base Controller Uses Mystica's Clips as Template
The base controller's states are populated with Mystica's animation clips (first character built). Override controllers replace them with character-specific clips.

**Rationale:** Unity's AnimatorOverrideController requires real clips with actual frame data in each state. Empty/zero-frame clips cause edge cases with exit time transitions (division by zero on clip length). Using real clips as the base means the controller works out of the box.

## File Plan

### Modified Files

| # | File | Changes |
|---|------|---------|
| 1 | `Editor/Animation/AnimationForgeMetadata.cs` | Add canonical state registry (attack_1–attack_10, block, guard, hurt, death). Add state-to-metadata mapping for validation. |
| 2 | `Scripts/Combat/Animation/TomatoFighterAnimatorParams.cs` | Add `ATTACK_1_TRIGGER`–`ATTACK_10_TRIGGER`, `BLOCKTRIGGER`, `GUARDTRIGGER` constants. Add `STATE_BLOCK`, `STATE_GUARD`, `STATE_HURT`, `STATE_DEATH` state name constants. |
| 3 | `Editor/Animation/AnimationBuilder.cs` | Refactor `BuildController()` to create base controller + override controllers. Add placeholder clip generation. Add validation (error on unmapped metadata anims, warning on missing canonical states). |

### New Files

| # | File | Purpose |
|---|------|---------|
| 4 | `Editor/Animation/AnimationEventStamper.cs` | Pipeline Step 3. Reads AttackData SOs, stamps Animation Events onto .anim clips at correct frame times. Menu: `TomatoFighters > Stamp Animation Events`. |

### Generated Outputs (not hand-edited)

- `Assets/Animations/Base/BaseCharacter_Controller.controller`
- `Assets/Animations/Brutor/Brutor_Override.overrideController`
- `Assets/Animations/Slasher/Slasher_Override.overrideController`
- `Assets/Animations/Mystica/Mystica_Override.overrideController`
- `Assets/Animations/Viper/Viper_Override.overrideController`
- Placeholder `.anim` clips (where art is missing)

## Execution Order

1. **AnimationForgeMetadata.cs** — add canonical state registry + validation data
2. **TomatoFighterAnimatorParams.cs** — add attack trigger + state constants
3. **AnimationBuilder.cs** — refactor to base + override pattern with placeholder generation + validation
4. **AnimationEventStamper.cs** — new pipeline step for baking events onto clips

## Pipeline Usage (after implementation)

```
# Full pipeline (run in order):
TomatoFighters > Import Sprite Sheets > All Characters    # Step 1: slice sprites
TomatoFighters > Build Animations > All Characters         # Step 2: clips + base controller + overrides
TomatoFighters > Stamp Animation Events                    # Step 3: bake hitbox/combo events

# After changing AttackData timing only:
TomatoFighters > Stamp Animation Events                    # Re-stamp without rebuilding clips
```

## Risks & Gotchas

1. **Character Creator scripts** reference controller paths — they need updating to load the override controller instead of `{Name}_Controller.controller`
2. **ComboDefinition assets** need their `animationTrigger` strings updated to match the new `attack_1`–`attack_10` naming convention
3. **CharacterAnimationBridge** should work unchanged — it only drives Speed/IsGrounded, which are on the base controller
4. **AttackData SOs may not exist yet** — AnimationEventStamper should gracefully skip characters without AttackData and log a warning
