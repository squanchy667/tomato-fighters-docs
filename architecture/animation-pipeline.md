# Animation Pipeline

## Overview

The animation system is a **data-driven, zero-hardcoded-names** pipeline that converts sprite sheet exports from **Animation Forge** (an external tool) into fully wired Unity AnimatorControllers. No code changes are needed to add new animations — drop a PNG, update `metadata.json`, run two menu commands.

```
Animation Forge (external tool)
    │
    ▼  metadata.json + sprite sheet PNGs
    │
┌───┴─────────────────────────────────────────────────────┐
│  Step 1: SpriteSheetImporter (Editor menu)              │
│  Reads metadata → configures TextureImporter →          │
│  grid-slices PNGs into named sprite frames              │
└───┬─────────────────────────────────────────────────────┘
    │  sliced Sprite assets
    ▼
┌───┴─────────────────────────────────────────────────────┐
│  Step 2: AnimationBuilder (Editor menu)                 │
│  Loads sliced sprites → creates .anim clips →           │
│  builds AnimatorController with locomotion + action     │
│  states, Speed-driven transitions + trigger-driven      │
│  action states                                          │
└───┬─────────────────────────────────────────────────────┘
    │  .anim clips + .controller
    ▼
┌─────────────────────────────────────────────────────────┐
│  Step 3: CharacterAnimationBridge (Runtime)             │
│  Reads CharacterMotor state each frame →                │
│  pushes Speed/IsGrounded to Animator                    │
│                                                         │
│  Step 4: ComboController (Runtime)                      │
│  Fires per-step animation triggers (AttackTrigger etc.) │
│  on combo input → drives action state transitions       │
└─────────────────────────────────────────────────────────┘
```

## Source Files

| File | Location | Type | Purpose |
|------|----------|------|---------|
| `AnimationForgeMetadata` | `Editor/Animation/` | Editor | Shared data model — parses `metadata.json` |
| `SpriteSheetImporter` | `Editor/Animation/` | Editor | Step 1 — imports and grid-slices sprite sheets |
| `AnimationBuilder` | `Editor/Animation/` | Editor | Step 2 — creates `.anim` clips + `AnimatorController` |
| `TomatoFighterAnimatorParams` | `Combat/Animation/` | Runtime | Shared string constants for Animator parameters |
| `CharacterAnimationBridge` | `Combat/Animation/` | Runtime | Step 3 — bridges motor state → Animator each frame |
| `ComboController` | `Combat/Combo/` | Runtime | Step 4 — fires attack triggers from combo state machine |

## Step 1: Sprite Sheet Import

**Menu:** `TomatoFighters > Import Sprite Sheets`

**Class:** `SpriteSheetImporter`

For each animation entry in `metadata.json`:

1. Locates the PNG at `Assets/animations/tomato_fighter_animations/Sprites/{character}_{animName}.png`
2. Configures `TextureImporter`: Sprite Mode = Multiple, PPU from metadata, uncompressed, max 8192px
3. Grid-slices into individual frames using `frame_w` / `frame_h` / `cols` / `rows`
4. Sets custom pivot (typically bottom-center `[0.5, 0.0]`)
5. Names frames with zero-padded indices (`idle_00`, `idle_01`, ...) so lexicographic sort = frame order

### Frame Layout

Frames are laid out **left-to-right, top-to-bottom** in the sheet. Unity sprite rects have origin at **bottom-left**, so the Y coordinate is flipped:

```
Y = texHeight - (row + 1) * frameHeight
```

> **Note:** Uses the deprecated `TextureImporter.spritesheet` API because `ISpriteEditorDataProvider` requires an assembly reference not exposed in Unity 6's built-in 2D Sprite module. Suppressed with `#pragma warning disable CS0618`.

## Step 2: Animation Building

**Menu:** `TomatoFighters > Build Animations` (run *after* Step 1)

**Class:** `AnimationBuilder`

**Output:** `Assets/Animations/TomatoFighter/` — `.anim` clips + `TomatoFighter_Controller.controller`

### Phase A — Create `.anim` Clips

- Loads sliced sprites from each sheet (sorted by name to preserve frame order)
- Creates `ObjectReferenceKeyframe` array with `time = frameIndex / fps`
- Binding path is `""` (empty) — Animator and SpriteRenderer share the same GameObject
- Sets `loopTime` from the `loop` field in metadata

### Phase B — Build AnimatorController

The `loop` field in metadata determines how each animation is wired:

| `loop` value | Category | Transition type |
|---|---|---|
| `true` | Locomotion | Speed float thresholds |
| `false` | Action | Trigger parameter, returns to idle via Exit Time |

#### Animator Parameters

| Parameter | Type | Default | Set by |
|-----------|------|---------|--------|
| `Speed` | Float | 0 | `CharacterAnimationBridge` |
| `IsGrounded` | Bool | true | `CharacterAnimationBridge` |
| `AttackTrigger` | Trigger | — | `ComboController` |
| `HurtTrigger` | Trigger | — | Reserved (T016+) |
| `DeathTrigger` | Trigger | — | Reserved (T016+) |
| `{animName}Trigger` | Trigger | — | Auto-generated per action animation |

#### Locomotion State Machine

```
         Speed > 0.1          Speed > 0.9
  idle ──────────────► walk ──────────────► run
  idle ◄──────────────  walk ◄──────────────  run
         Speed < 0.1          Speed < 0.9

  run ──────────────► idle    (direct: Speed < 0.1)
```

- **Default state:** idle
- All transitions: `hasExitTime = false`, `duration = 0.05s`
- Speed thresholds: `idle < 0.1 < walk < 0.9 < run`

#### Action States

- **Entry:** `AnyState → action` via trigger (instant, `hasExitTime = false`, `duration = 0`)
- **Exit:** `action → idle` via Exit Time = 1.0 (`duration = 0.05s`) — clip plays once, then returns

## Step 3: Runtime — Locomotion Bridge

**Class:** `CharacterAnimationBridge`

Runs in `LateUpdate` (after motor state is settled). Maps **player intent** to Speed values:

| Condition | Speed value | Animator state |
|-----------|-------------|----------------|
| No movement input | `0.0` | idle |
| Moving, not running | `0.5` | walk |
| Moving + run active | `1.0` | run |

**Intent-driven, not velocity-based** — animation follows input intent, not physics velocity magnitude. This gives snappy state transitions appropriate for a 2D beat 'em up.

**Run behaviour:** Run is sticky — pressing Left Ctrl while moving activates it, releasing Ctrl keeps running. Run resets on: stop, attack, dash, jump (handled in `CharacterMotor.RequestRun`).

**Why a separate component?** Keeps `CharacterMotor` free of Animator dependencies so the motor remains unit-testable with plain C# tests.

Uses cached `Animator.StringToHash()` to avoid string lookups every frame.

## Step 4: Runtime — Combat Animation Triggers

**Class:** `ComboController`

When the `ComboStateMachine` fires a step, `ComboController.TriggerStepAnimation()` reads the `animationTrigger` string from the current `ComboStep` and calls `Animator.SetTrigger()`. This drives the action state transitions built in Step 2.

**Animation events** on attack clips call back into `ComboController`:
- `OnComboWindowOpen()` — opens the cancel/combo input window
- `OnFinisherEnd()` — signals the finisher is complete

This creates a **bidirectional relationship**: ComboController tells the Animator *when* to play an attack, and the Animator tells ComboController *when* the attack reaches key frames via animation events.

## metadata.json Reference

```json
{
  "character_name": "tomato_fighter",
  "animations": {
    "idle": {
      "frame_w": 464,
      "frame_h": 688,
      "cols": 8,
      "rows": 8,
      "n_frames": 59,
      "fps": 24,
      "loop": true,
      "pivot": [0.5, 0.0],
      "ppu": 256
    }
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `character_name` | string | Used to locate sprite sheets: `{character}_{animName}.png` |
| `frame_w` / `frame_h` | int | Frame dimensions in pixels |
| `cols` / `rows` | int | Grid layout of frames in the sheet |
| `n_frames` | int | Total frame count (may be less than `cols * rows`) |
| `fps` | int | Playback frame rate |
| `loop` | bool | `true` = locomotion state, `false` = action state |
| `pivot` | float[2] | Normalized pivot point (0-1). `[0.5, 0.0]` = bottom-center |
| `ppu` | int | Pixels per unit for the sprite |

## How to Add a New Animation

1. Export the sprite sheet PNG from your art tool
2. Place it at `Assets/animations/tomato_fighter_animations/Sprites/{character}_{animName}.png`
3. Add the entry to `metadata.json`:
   - Set `loop: true` for locomotion animations (idle, walk, run, etc.)
   - Set `loop: false` for action animations (attacks, hurt, death, etc.)
4. In Unity: **TomatoFighters > Import Sprite Sheets**
5. In Unity: **TomatoFighters > Build Animations**

No code changes needed — the pipeline picks it up automatically.

## Future Considerations

- **Multi-character support:** Output paths are currently hardcoded to `TomatoFighter`. When adding Brutor/Slasher/Mystica/Viper, the pipeline will need parameterized output paths (one controller per character).
- **Combo chaining:** Action states currently return to idle after playing. Chained combos (light1 → light2 → light3) rely on the ComboController firing the next trigger before the exit-time transition completes.
- **Hit reaction / death:** `HurtTrigger` and `DeathTrigger` are reserved in `TomatoFighterAnimatorParams` for T016+.
