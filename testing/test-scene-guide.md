# Test Scene Guide — MovementTest

## Overview

**MovementTest** is the primary playtest scene for validating movement, combos, and animations. It provides a contained 2D arena with a fully-wired Player prefab using the **Mystica** character archetype by default.

**Scene path:** `Assets/Scenes/MovementTest.unity`

---

## Quick Start

1. **Open** `Assets/Scenes/MovementTest.unity` in Unity Editor
2. **Press Play**
3. Use the controls below to test movement and combat

> If the scene or prefab is missing, regenerate them via the Editor menu (see [Editor Tooling](#editor-tooling) below).

---

## Controls

| Input | Action | Notes |
|-------|--------|-------|
| **WASD** | Move | 8-directional on the ground plane |
| **Space** | Jump | Supports buffering (0.12s) and coyote time (0.1s) |
| **Left Shift** | Dash | Directional, has i-frames, 0.5s cooldown |
| **Left Ctrl** | Run | Sticky toggle — 1.5x speed, releases on zero input |
| **Left Mouse** | Light Attack | Chains: Light1 → Light2 → LightFinisher |
| **C** | Heavy Attack | Chains: Heavy1 → HeavyFinisher |
| **Gamepad Left Stick** | Move | Same as WASD |
| **Arrow Keys** | Move | Same as WASD |

### Combo Flow

```
Light chain:   Light1 (1.0x) → Light2 (1.1x) → LightFinisher (1.4x)
Heavy chain:   Heavy1 (1.8x) → HeavyFinisher (2.5x)
```

- **Combo window:** 0.5s default (0.6s on Heavy1)
- **Cancel buffer:** 0.15s — dash or jump can cancel an active attack after hit confirm
- Light2 supports dash cancel on hit; Heavy1 supports dash + jump cancel on hit

---

## Scene Layout

```
┌──────────────────────────────────────────┐
│  [Control Hints - TextMesh]              │  ← Top of screen
│                                          │
│  ┌──────── 20 × 10 Arena ─────────────┐ │
│  │  - - - - - - - - - - - - - - - - - │ │  ← 9 horizontal grid lines
│  │  - - - - - - - - - - - - - - - - - │ │    (depth perception)
│  │                                     │ │
│  │            🍅 Player                │ │
│  │           (shadow)                  │ │
│  │  - - - - - - - - - - - - - - - - - │ │
│  │  - - - - - - - - - - - - - - - - - │ │
│  └─────────────────────────────────────┘ │
│  [4 invisible wall colliders]            │
└──────────────────────────────────────────┘
```

- **Camera:** Orthographic, size 7, dark background (0.15, 0.15, 0.2)
- **Ground:** Dark green sprite (0.25, 0.3, 0.2) with grid lines
- **Walls:** 4 invisible BoxCollider2D (1-unit thick) bounding the arena

---

## Player Prefab

**Path:** `Assets/Prefabs/Player/Player.prefab`

### Hierarchy

```
Player (Root)
│   Rigidbody2D     — Dynamic, gravity=0, continuous collision, freeze-rot-Z
│   BoxCollider2D   — Size 0.8 × 0.6
│   CharacterMotor
│   ComboController
│   ComboDebugUI
│   CharacterAnimationBridge
│   CharacterInputHandler
│
├── Sprite (child)
│   │   SpriteRenderer  — tomato_fighter_idle, sorting order 1
│   └── Animator         — TomatoFighter_Controller
│
└── Shadow (child)
        SpriteRenderer  — Black @ 30% alpha, sorting order 0
```

### Default Configuration

| Component | Config Asset | Character |
|-----------|-------------|-----------|
| CharacterMotor | `ScriptableObjects/MovementConfigs/Mystica_MovementConfig` | Mystica |
| ComboController | `ScriptableObjects/ComboDefinitions/Mystica_ComboDefinition` | Mystica |

### Key Movement Parameters (Mystica)

| Parameter | Value | Description |
|-----------|-------|-------------|
| moveSpeed | 8 | Base horizontal speed (units/sec) |
| depthSpeed | 5 | Y-axis speed (slower for belt-scroll feel) |
| groundAcceleration | 65 | Snappy ground response |
| airAcceleration | 35 | Reduced air control |
| jumpForce | 14 | Initial jump velocity |
| jumpGravity | 38 | Custom gravity (lighter = floatier) |
| dashSpeed | 20 | Dash velocity |
| dashDuration | 0.14s | Short, snappy |
| dashCooldown | 0.5s | Cooldown between dashes |
| dashHasIFrames | true | Invincible during dash |
| runSpeedMultiplier | 1.5x | Run mode speed boost |

---

## Physics Model

The scene uses a **belt-scroll** physics model (like classic beat 'em ups):

- **Rigidbody2D gravity = 0** — no Unity gravity
- **Jump** is simulated via a manual jump arc: the Sprite child is offset vertically by `jumpHeight` each frame
- **Ground movement** uses Rigidbody2D velocity on the X/Y plane
- **Knockback** will use Rigidbody2D forces (not transform.position)

### Movement States

```
MovementStateMachine: Grounded ←→ Airborne ←→ Dashing

Grounded  — Full movement, can jump, can dash
Airborne  — Reduced acceleration, can't dash
Dashing   — Override velocity, i-frames active, movement locked
```

---

## Animation Setup

**Animator Controller:** `Assets/animations/TomatoFighter/TomatoFighter_Controller.controller`

### Parameters

| Parameter | Type | Purpose |
|-----------|------|---------|
| Speed | float | Drives idle/walk/run blend |
| IsGrounded | bool | Ground vs air state |
| Light1 | trigger | 1st light attack |
| Light2 | trigger | 2nd light attack |
| LightFinisher | trigger | 3rd light (finisher) |
| Heavy1 | trigger | 1st heavy attack |
| HeavyFinisher | trigger | Heavy finisher |

### Sprite Sheets

| Animation | Frames | FPS | Loop |
|-----------|--------|-----|------|
| Idle | 50 | 12 | Yes |
| Walk | 36 | 14 | Yes |
| Run | 36 | 16 | Yes |
| Jump | 31 | 16 | No |
| Land | 20 | 16 | No |

**Location:** `Assets/animations/tomato_fighter_animations/Sprites/`

---

## Editor Tooling

Two menu tools regenerate the scene and prefab from scratch:

### Create Player Prefab

**Menu:** `TomatoFighters > Create Player Prefab`
**Script:** `Assets/Editor/Prefabs/PlayerPrefabCreator.cs`

Builds `Player.prefab` with all components, auto-creates `Mystica_MovementConfig` and `Mystica_ComboDefinition` if missing.

### Create Movement Test Scene

**Menu:** `TomatoFighters > Create Movement Test Scene`
**Script:** `Assets/Editor/Prefabs/MovementTestSceneCreator.cs`

Builds `MovementTest.unity` with camera, arena, walls, grid lines, player instance, and control hints.

> **InputActionReference workaround:** Unity's `InputActionReference.Create()` doesn't survive prefab serialization. The scene creator re-wires all input references at scene creation time by loading the `InputActionAsset` and reconnecting each action by path (e.g. `Player/Move`).

---

## Switching Characters

The Player prefab defaults to Mystica. To test a different character:

1. Select the **Player** GameObject in the scene hierarchy
2. On **CharacterMotor**: swap `Config` to another MovementConfig (e.g. `Brutor_MovementConfig`)
3. On **ComboController**: swap `Combo Definition` to the matching ComboDefinition
4. On both components: set `Character Type` to match

Available configs:
- `ScriptableObjects/MovementConfigs/Brutor_MovementConfig`
- `ScriptableObjects/MovementConfigs/Mystica_MovementConfig`
- `ScriptableObjects/ComboDefinitions/Brutor_ComboDefinition`
- `ScriptableObjects/ComboDefinitions/Mystica_ComboDefinition`

---

## What to Test

### Movement Checklist

- [ ] 8-directional movement feels responsive (WASD)
- [ ] Depth movement (Y-axis) is noticeably slower than horizontal
- [ ] Jump arc feels good — not too floaty, not too snappy
- [ ] Jump buffering works (press Space slightly before landing)
- [ ] Coyote time works (jump briefly after walking off edge)
- [ ] Dash is short and snappy with clear direction
- [ ] Dash cooldown prevents spam
- [ ] Run mode activates/deactivates cleanly
- [ ] Player stays within arena walls

### Combo Checklist

- [ ] Light chain progresses: Light1 → Light2 → LightFinisher
- [ ] Heavy chain progresses: Heavy1 → HeavyFinisher
- [ ] Combo drops after window expires (0.5s of no input)
- [ ] Dash cancel works on Light2 and Heavy1 after hit
- [ ] Jump cancel works on Heavy1 after hit
- [ ] Attack locks movement during animation
- [ ] Finisher plays fully before returning to idle

### Animation Checklist

- [ ] Idle animation plays when stationary
- [ ] Walk/Run animations blend based on speed
- [ ] Jump and Land animations trigger correctly
- [ ] Attack triggers fire the correct animations
- [ ] No animation glitches on rapid state transitions

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Scene is empty / Player missing | Run `TomatoFighters > Create Movement Test Scene` |
| Player doesn't respond to input | Check that `InputSystem_Actions.inputactions` exists and the Input Handler references are wired |
| Player falls through floor | Verify Rigidbody2D gravity scale = 0 |
| Animations don't play | Check Animator Controller is assigned on the Sprite child |
| Combo window advances instantly | ComboDebugUI auto-advances in Editor when no attack animations exist — this is intentional for testing without animations |
| Player clips through walls | Ensure wall colliders exist (regenerate scene if needed) |
| Input references are null at runtime | Re-run `Create Movement Test Scene` — this re-wires InputActionReferences |
