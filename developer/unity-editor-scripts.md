# Unity Editor Scripts

> Location: `unity/TomatoFighters/Assets/Editor/`
> All scripts are idempotent — safe to re-run without duplicating work.

---

## Quick Reference

| # | Script | Menu Path | Creates / Modifies |
|---|--------|-----------|-------------------|
| 1 | Import Sprite Sheets | `TomatoFighters > Import Sprite Sheets` | Configures PNG import settings, grid-slices sprite sheets into individual frames |
| 2 | Build Animations | `TomatoFighters > Build Animations` | `.anim` clips + `AnimatorController` with locomotion/action/airborne states |
| 3 | Create Player Prefab | `TomatoFighters > Create Player Prefab` | `Player.prefab` with all components wired (motor, combo, input, animator) |
| 4 | Create Movement Test Scene | `TomatoFighters > Create Movement Test Scene` | `MovementTest.unity` scene with camera, arena, walls, player, enemy, and DamagePipelineDiagnostic |
| 5 | Create Mystica Attacks | `Tools > TomatoFighters > Create Mystica Attacks` | 4 AttackData assets (Strike 1-3, Arcane Bolt) |
| 6 | Create All Character Attacks | `Tools > TomatoFighters > Create All Character Attacks` | 22 AttackData assets (Brutor 7, Slasher 8, Viper 6, Mystica extra 1) |
| 7 | Create Combo Definitions | `Tools > TomatoFighters > Create Combo Definitions` | 4 ComboDefinition + 4 ComboInteractionConfig assets, wires Player prefab |
| 8 | Create All Path Assets | `TomatoFighters > Create All Path Assets` | 12 PathData assets (4 characters x 3 paths) |
| 9 | Assign All Hitbox IDs | `Tools > TomatoFighters > Assign All Hitbox IDs` | Sets `hitboxId` field on all 26 AttackData assets |
| 10 | Setup Mystica Character | `Tools > TomatoFighters > Setup Mystica Character` | Hitbox children on prefab, HitboxManager component, Mystica hitboxId wiring |
| 11 | Create TestDummy Prefab | `TomatoFighters > Create TestDummy Prefab` | `TestDummy.prefab` + EnemyData SO + DummyPunch AttackData SO + WhiteSquare debug sprite |

---

## Workflow Recipes

### New Animation GIFs Arrive (from Animation Forge)

**When:** The artist delivers a new set of sprite sheet PNGs + updated `metadata.json` from Animation Forge.

**Steps:**

1. Drop PNG files into `Assets/animations/tomato_fighter_animations/Sprites/`
2. Replace `Assets/animations/tomato_fighter_animations/metadata.json` with the updated one
3. Run **#1 Import Sprite Sheets** — slices all PNGs into individual sprite frames
4. Run **#2 Build Animations** — creates `.anim` clips and rebuilds the `AnimatorController`
5. Run **#3 Create Player Prefab** — re-wires the Animator on the Player prefab with the updated controller
6. Run **#4 Create Movement Test Scene** — rebuilds the test scene with the updated prefab

> Scripts #1 and #2 are driven entirely by `metadata.json`. New animations (e.g., `dash`, `hurt`, `death`) are picked up automatically — just add the PNG and its entry in metadata.json.

---

### Fresh Project Setup (from scratch)

**When:** Cloning the repo for the first time, or rebuilding all assets from clean state.

**Steps:**

```
1. Import Sprite Sheets               ← slice PNGs into sprite frames
2. Build Animations                    ← create .anim clips + AnimatorController
3. Create Player Prefab                ← Player.prefab with motor, combo, input, animator
4. Create Mystica Attacks              ← 4 Mystica AttackData assets
5. Create All Character Attacks        ← 22 more AttackData assets (all characters)
6. Create Combo Definitions            ← 4 ComboDefinitions + 4 InteractionConfigs + prefab wiring
7. Create All Path Assets              ← 12 PathData assets
8. Assign All Hitbox IDs               ← hitboxId on all 26 attacks
9. Setup Mystica Character             ← hitbox children + HitboxManager on prefab (select Player root first)
10. Create Movement Test Scene         ← test scene with arena + player + enemy
11. Create TestDummy Prefab             ← TestDummy.prefab with enemy + hitbox + debug visuals
```

---

### Attack Balance Changed

**When:** Designers change damage multipliers, knockback values, hitbox frames, or combo flow on existing attacks.

**Steps:**

1. Edit the values in `CreateAllCharacterAttacks.cs` (or `CreateMysticaAttacks.cs` for Mystica's base 4)
2. **Delete** the existing `.asset` files in `ScriptableObjects/Attacks/{Character}/` (the scripts skip existing assets)
3. Run **#5 Create Mystica Attacks** and/or **#6 Create All Character Attacks**
4. Run **#7 Create Combo Definitions** — rebuilds combo trees with updated AttackData refs
5. Run **#9 Assign All Hitbox IDs** — re-apply hitboxId mappings

---

### Combo Structure Changed

**When:** Designers add a new attack to a combo chain, change branching, or modify finishers.

**Steps:**

1. If it's a new attack: add it in `CreateAllCharacterAttacks.cs`, delete the old asset, and re-run **#6**
2. Edit the combo tree in `CreateComboDefinitions.cs` (step indices, nextOnLight/nextOnHeavy, cancel flags)
3. Run **#7 Create Combo Definitions** — overwrites existing ComboDefinition assets (preserves GUIDs)

---

### Adding a New Character

**When:** A new playable character is introduced (e.g., 5th character).

**Steps:**

1. Add the character's entries to `CreateAllCharacterAttacks.cs`
2. Add the character's combo in `CreateComboDefinitions.cs`
3. Add the character's 3 paths in `PathDataCreator.cs`
4. Add hitbox mappings in `AssignAllHitboxIds.cs`
5. Create a `Setup{Character}Character.cs` following the Mystica pattern (see [Adding Character Setup Scripts](#adding-new-character-setup-scripts))
6. Run scripts **#6 → #7 → #8 → #9 → new setup script**

---

### New Path/Upgrade Balance Changed

**When:** Designers change stat bonuses or ability IDs for character upgrade paths.

**Steps:**

1. Edit the values in `PathDataCreator.cs`
2. Run **#8 Create All Path Assets** — overwrites existing PathData assets in place

---

## Detailed Script Reference

### #1 — Import Sprite Sheets

| | |
|---|---|
| **Menu** | `TomatoFighters > Import Sprite Sheets` |
| **File** | `Editor/Animation/SpriteSheetImporter.cs` |
| **Input** | `metadata.json` + PNG sprite sheets in `animations/tomato_fighter_animations/Sprites/` |
| **Output** | Configured TextureImporter settings + grid-sliced sprite frames on each PNG |
| **Prereqs** | PNGs and `metadata.json` from Animation Forge exist in the Sprites folder |

**What it creates:**

For each animation entry in `metadata.json`:
- Configures the PNG's TextureImporter: Sprite Mode = Multiple, PPU from metadata, uncompressed, max 8192px
- Grid-slices into individual frames using `frame_w` / `frame_h` / `cols` / `rows`
- Names frames with zero-padded indices (`idle_00`, `idle_01`, ...) for correct sort order
- Sets custom pivot (typically bottom-center) per metadata

**Picks up new animations automatically** — any new entry in `metadata.json` with a matching PNG gets processed.

---

### #2 — Build Animations

| | |
|---|---|
| **Menu** | `TomatoFighters > Build Animations` |
| **File** | `Editor/Animation/AnimationBuilder.cs` |
| **Input** | Sliced sprites (from #1) + `metadata.json` |
| **Output** | `.anim` clips + `TomatoFighter_Controller.controller` in `Assets/Animations/TomatoFighter/` |
| **Prereqs** | Run Import Sprite Sheets (#1) first |

**What it creates:**

- One `.anim` clip per animation (named `tomato_fighter_{animName}.anim`)
- An `AnimatorController` with three categories of states:

| Category | Driven By | Animations | Behavior |
|----------|-----------|------------|----------|
| **Locomotion** (`loop: true`) | `Speed` float | idle, walk, run | Speed thresholds: idle↔walk at 0.1, walk↔run at 0.9 |
| **Airborne** (jump/land) | `IsGrounded` bool | jump, land | Locomotion→jump when `!IsGrounded`, jump→land when `IsGrounded`, land→idle on exit |
| **Action** (`loop: false`, not airborne) | `{name}Trigger` trigger | dash, attack anims, etc. | AnyState→action via trigger, action→idle on exit time |

- Adds animator parameters: `Speed` (float), `IsGrounded` (bool, default true), plus one trigger per action animation
- **Replaces** the existing controller — always rebuilt clean

---

### #3 — Create Player Prefab

| | |
|---|---|
| **Menu** | `TomatoFighters > Create Player Prefab` |
| **File** | `Editor/Prefabs/PlayerPrefabCreator.cs` |
| **Input** | AnimatorController, MovementConfig, ComboDefinition, InputActionAsset |
| **Output** | `Assets/Prefabs/Player/Player.prefab` |
| **Prereqs** | Run Build Animations (#2) first for the controller. Creates MovementConfig and ComboDefinition placeholders if missing |

**What it creates:**

A Player prefab with this structure:

```
Player (root)
├── Rigidbody2D (gravity=0, continuous, interpolate)
├── BoxCollider2D (0.8 x 0.6)
├── CharacterMotor → wired to MovementConfig (Mystica)
├── ComboController → wired to ComboDefinition + Animator + Motor
├── ComboDebugUI
├── CharacterAnimationBridge → wired to Animator + Motor
├── CharacterInputHandler → wired to Motor + ComboController + input actions
├── Sprite (child)
│   ├── SpriteRenderer (sorting order 1, first idle frame)
│   └── Animator → TomatoFighter_Controller
└── Shadow (child)
    └── SpriteRenderer (black, 30% alpha, sorting order 0)
```

- Creates `Mystica_MovementConfig` asset if missing (move=8, jump=14, dash=20, etc.)
- Creates placeholder `Mystica_ComboDefinition` if missing
- Wires all input actions: Move=WASD, Jump=Space, Dash=L-Shift, Light=LMB, Heavy=C, Run=L-Ctrl
- **Safe to re-run** — loads existing prefab and updates components, preserving manually added children

---

### #4 — Create Movement Test Scene

| | |
|---|---|
| **Menu** | `TomatoFighters > Create Movement Test Scene` |
| **File** | `Editor/Prefabs/MovementTestSceneCreator.cs` |
| **Input** | `Player.prefab`, `InputSystem_Actions.inputactions` |
| **Output** | `Assets/Scenes/MovementTest.unity` |
| **Prereqs** | Run Create Player Prefab (#3) first. Falls back to inline player if prefab missing |

**What it creates:**

- Orthographic camera (size 7, dark background)
- 20x10 arena with 4 invisible wall colliders
- Dark green ground plane with grid lines for depth perception
- Player instance from prefab with all input actions re-wired
- Controls hint text at top of screen
- **DamagePipelineDiagnostic** component on Main Camera (temporary debug tool, added via `Type.GetType()` string lookup so it compiles even if the diagnostic script is removed). Validates layers, colliders, component wiring, and Physics2D settings on scene Start.

**Why re-wiring?** `InputActionReference.Create()` doesn't survive prefab serialization, so input actions are wired on the scene instance directly.

---

### #5 — Create Mystica Attacks

| | |
|---|---|
| **Menu** | `Tools > TomatoFighters > Create Mystica Attacks` |
| **File** | `Editor/CreateMysticaAttacks.cs` |
| **Output** | 4 AttackData assets in `ScriptableObjects/Attacks/Mystica/` |

**What it creates:**

| Asset | Attack Name | Damage | Knockback | Hitbox Frames |
|-------|-------------|--------|-----------|---------------|
| MysticaStrike1 | Magic Burst 1 | 0.6x | (1.5, 0) | start 3, active 4, total 16 |
| MysticaStrike2 | Magic Burst 2 | 0.8x | (2.0, 0.3) | start 3, active 4, total 18 |
| MysticaStrike3 | Magic Burst 3 | 1.0x | (3.0, 0.5) | start 4, active 5, total 22 |
| MysticaArcaneBolt | Arcane Bolt | 1.4x | (2.0, 1.0) | start 6, active 6, total 30 |

**Skips** assets that already exist. Delete the `.asset` file first to regenerate.

---

### #6 — Create All Character Attacks

| | |
|---|---|
| **Menu** | `Tools > TomatoFighters > Create All Character Attacks` |
| **File** | `Editor/CreateAllCharacterAttacks.cs` |
| **Output** | 22 AttackData assets in `ScriptableObjects/Attacks/{Character}/` |

**What it creates:**

| Character | # Attacks | Attacks |
|-----------|-----------|---------|
| **Brutor** (7) | Shield Bash 1 & 2, Sweep, Launcher, Air Slam, Overhead Slam, Ground Pound |
| **Slasher** (8) | Slash 1-3, Spin Finisher, Lunge, Heavy Slash, Piercing Lunge, Quick Re-entry |
| **Viper** (6) | Shot 1 & 2, Rapid Burst, Quick Charged, Charged Shot, Piercing Shot |
| **Mystica** (1) | Empowered Bolt (the 5th attack not in Create Mystica Attacks) |

**Skips** assets that already exist. Configures all combat properties: damage multiplier, knockback, launch force, hitbox frame timing, telegraph type, and special flags (wall bounce, launch, OTG, air attack).

---

### #7 — Create Combo Definitions

| | |
|---|---|
| **Menu** | `Tools > TomatoFighters > Create Combo Definitions` |
| **File** | `Editor/CreateComboDefinitions.cs` |
| **Output** | 4 ComboDefinition assets + 4 ComboInteractionConfig assets + Player prefab wiring |
| **Prereqs** | AttackData assets exist for all characters (run #5 + #6 first) |

**What it creates:**

**ComboDefinition assets** (in `ScriptableObjects/ComboDefinitions/`):

| Character | Steps | Light Chain | Heavy Chain | Branches |
|-----------|-------|-------------|-------------|----------|
| Brutor | 7 | Bash1→Bash2→Sweep(F) | Overhead→GroundPound(F) | L1→H: Launcher→AirSlam(F) |
| Slasher | 8 | Slash1→2→3→Spin(F) | Heavy→PiercingLunge(F) | L2→H: Lunge; H1→L: QuickSlash→re-enter L2 |
| Mystica | 5 | Strike1→2→3(F) | ArcaneBolt→EmpoweredBolt(F) | — |
| Viper | 6 | Shot1→2→RapidBurst(F) | Charged→Piercing(F) | L2→H: QuickCharged |

**ComboInteractionConfig assets** (in `ScriptableObjects/ComboInteractionConfigs/`):
- Per-character cancel priorities, dash/jump cancel behavior, stagger/death resets, movement locks

**Prefab auto-wiring:** Reads which ComboDefinition is assigned on the Player prefab and wires the matching ComboInteractionConfig.

**Overwrites** existing assets (preserves GUIDs). Errors if AttackData assets are missing.

---

### #8 — Create All Path Assets

| | |
|---|---|
| **Menu** | `TomatoFighters > Create All Path Assets` |
| **File** | `Editor/PathDataCreator.cs` |
| **Output** | 12 PathData assets in `ScriptableObjects/Paths/{Character}/` |

**What it creates:**

| Character | Path 1 | Path 2 | Path 3 |
|-----------|--------|--------|--------|
| **Brutor** | Warden (aggro/taunt) | Bulwark (personal defense) | Guardian (team defense) |
| **Slasher** | Executioner (single-target burst) | Reaper (AOE/cleave) | Shadow (evasion/i-frames) |
| **Mystica** | Sage (healer) | Enchanter (buffer) | Conjurer (summoner) |
| **Viper** | Marksman (ranged DPS) | Trapper (crowd control) | Arcanist (mana attacks) |

Each path has 3 tiers with stat bonuses (HP, ATK, DEF, SPD, MNA, crit, stun, mana regen) and ability IDs.

**Overwrites** existing assets — safe to re-run after changing values.

---

### #9 — Assign All Hitbox IDs

| | |
|---|---|
| **Menu** | `Tools > TomatoFighters > Assign All Hitbox IDs` |
| **File** | `Editor/AssignAllHitboxIds.cs` |
| **Output** | Sets `hitboxId` field on all 26 AttackData assets |
| **Prereqs** | AttackData assets exist (run #5 + #6 first) |

**What it creates:**

Maps each attack to a reusable hitbox shape name:

| hitboxId | Shape | Attacks |
|----------|-------|---------|
| `Jab` | Small melee | Brutor Bash 1 & 2, Slasher Slash 1 & 2, Slasher Quick Re-entry |
| `Sweep` | Wide horizontal | Brutor Sweep, Slasher Cross Slash, Slasher Heavy Slash, Viper Rapid Burst |
| `Uppercut` | Tall vertical | Brutor Launcher |
| `Slam` | Circular impact | Brutor Air/Overhead Slam, Ground Pound, Slasher Spin Finisher |
| `Lunge` | Extended narrow | Slasher Lunge & Piercing Lunge, all Viper shots |
| `Burst` | Magic circle | Mystica Strike 1 & 2 |
| `BigBurst` | Magic circle (wide) | Mystica Strike 3 |
| `Bolt` | Magic bolt | Mystica Arcane Bolt & Empowered Bolt |

No prefab selection needed — operates on assets directly.

---

### #10 — Setup Mystica Character

| | |
|---|---|
| **Menu** | `Tools > TomatoFighters > Setup Mystica Character` |
| **File** | `Editor/SetupMysticaCharacter.cs` |
| **Output** | Hitbox children on Player prefab, HitboxManager component, hitboxId on 5 Mystica attacks |
| **Prereqs** | `PlayerHitbox` layer exists, Mystica AttackData assets exist, Player prefab open in prefab mode |

**What it creates:**

1. **3 hitbox child GameObjects** on the Player prefab:

   | Child | Collider | Size | Offset | Used By |
   |-------|----------|------|--------|---------|
   | `Hitbox_Burst` | CircleCollider2D | r=0.5 | (0.5, 0.1) | Magic Burst 1 & 2 |
   | `Hitbox_BigBurst` | CircleCollider2D | r=0.8 | (0.4, 0.1) | Magic Burst 3 (finisher) |
   | `Hitbox_Bolt` | BoxCollider2D | 1.6x0.35 | (1.1, 0.15) | Arcane Bolt & Empowered Bolt |

   Each child: has `HitboxDamage` component, starts **disabled**, on `PlayerHitbox` layer.

2. **HitboxManager component** on root — wires `ComboController` ref, sets `baseAttack = 10`

3. **hitboxId assignment** on 5 Mystica AttackData assets (Strike1→Burst, Strike2→Burst, Strike3→BigBurst, ArcaneBolt→Bolt, EmpoweredBolt→Bolt)

**How to run:**
1. Open `Prefabs/Player/Player.prefab` in prefab mode
2. Select the **Player root** in Hierarchy
3. Run the menu command
4. Save the prefab (Ctrl+S)

---

## Shared: AnimationForgeMetadata

| | |
|---|---|
| **File** | `Editor/Animation/AnimationForgeMetadata.cs` |
| **Not a menu item** | Shared data model used by #1 and #2 |

Parses the `metadata.json` file exported by Animation Forge. Contains:
- `MetadataRoot` — character name + dictionary of animations
- `AnimationEntry` — frame dimensions, grid layout, FPS, loop flag, pivot, PPU

**Key convention:** The `loop` field drives the entire animation wiring strategy:
- `true` → locomotion state (Speed-driven float transitions)
- `false` → action state (trigger-driven, returns to idle via exit time)
- `jump`/`land` names → airborne states (IsGrounded-driven, special-cased)

**metadata.json format:**
```json
{
  "character_name": "purple_mage",
  "animations": {
    "idle": { "frame_w": 128, "frame_h": 192, "cols": 4, "rows": 4, "n_frames": 17, "fps": 12, "loop": true, "pivot": [0.5, 0.0], "ppu": 128 },
    "walk": { "..." },
    "run":  { "..." },
    "jump": { "...loop: false..." },
    "land": { "...loop: false..." },
    "dash": { "...loop: false..." }
  }
}
```

---

## Dependency Graph

```
metadata.json + PNGs
       │
       ▼
  ┌─────────────────┐
  │ #1 Import Sprite │
  │    Sheets        │
  └────────┬────────┘
           ▼
  ┌─────────────────┐    ┌─────────────────┐    ┌────────────────┐
  │ #2 Build        │    │ #5 Create       │    │ #8 Create All  │
  │    Animations   │    │   Mystica Atks  │    │   Path Assets  │
  └────────┬────────┘    └───────┬─────────┘    └────────────────┘
           │                     │
           ▼                     ▼
  ┌─────────────────┐    ┌─────────────────┐
  │ #3 Create       │    │ #6 Create All   │
  │   Player Prefab │    │   Char Attacks  │
  └────────┬────────┘    └───────┬─────────┘
           │                     │
           │              ┌──────┴──────┐
           │              ▼             ▼
           │     ┌──────────────┐ ┌───────────────┐
           │     │ #7 Create    │ │ #9 Assign All │
           │     │  Combo Defs  │ │   Hitbox IDs  │
           │     └──────────────┘ └───────────────┘
           │
           ▼
  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
  │ #4 Create Test  │    │ #10 Setup       │    │ #11 Create      │
  │    Scene        │    │   Mystica Char  │    │  TestDummy Pref │
  └─────────────────┘    └─────────────────┘    └─────────────────┘
  (also spawns enemy)    (needs prefab open     (needs EnemyHurtbox
                          + PlayerHitbox layer)  + EnemyHitbox layers)
```

---

### #11 — Create TestDummy Prefab

| | |
|---|---|
| **Menu** | `TomatoFighters > Create TestDummy Prefab` |
| **File** | `Editor/Prefabs/TestDummyPrefabCreator.cs` |
| **Output** | `Assets/Prefabs/Enemies/TestDummy.prefab` + 2 SOs + debug sprite |
| **Prereqs** | `EnemyHurtbox` and `EnemyHitbox` layers exist in Project Settings |

**What it creates:**

1. **TestDummy prefab** at `Prefabs/Enemies/TestDummy.prefab`:

   ```
   TestDummy (root, EnemyHurtbox layer)
   ├── Rigidbody2D (gravity=0, continuous, interpolate)
   ├── BoxCollider2D (0.8x1.2, body hurtbox)
   ├── TestDummyEnemy component (wired to EnemyData + AttackData)
   ├── DebugHealthBar component (red fill, offset 1.2 above)
   ├── Sprite (child)
   │   └── SpriteRenderer (WhiteSquare, orange tint, 0.8x1.2 scale)
   └── Hitbox_Punch (child, EnemyHitbox layer, starts disabled)
       ├── BoxCollider2D (trigger, 0.8x0.6, offset -0.6x0.1)
       ├── HitboxDamage component (from Shared)
       └── DebugVisual (child)
           └── SpriteRenderer (WhiteSquare, semi-transparent red)
   ```

2. **TestDummy_EnemyData** SO at `ScriptableObjects/Enemies/`:
   - HP 100, pressure threshold 50, stun 2s, invuln 1s, knockback resist 0.3, speed 0

3. **DummyPunch AttackData** SO at `ScriptableObjects/Attacks/Enemy/`:
   - 0.5x damage, knockback (3, 0), hitbox start 2 / active 3 / total 15 frames

4. **WhiteSquare.png** at `Assets/Sprites/Debug/`:
   - 10x10 white PNG, 10 PPU, Point filter — used as base sprite for body and hitbox visuals

**Safe to re-run** — loads existing assets/prefab and updates in place.

---

## Adding New Character Setup Scripts

When building setup scripts for Brutor, Slasher, or Viper, follow `SetupMysticaCharacter.cs`:

1. **Copy** `SetupMysticaCharacter.cs` → `Setup{Character}Character.cs`
2. **Update** class name and menu path
3. **Define hitbox children** — shapes/sizes for that character's attacks. Reuse existing hitboxId names where the shape fits:

| Character | Expected Shapes | Notes |
|-----------|-----------------|-------|
| **Brutor** | Jab, Sweep, Uppercut, Slam | Heavy, close-range |
| **Slasher** | Jab, Sweep, Slam, Lunge | Fast, medium-range |
| **Viper** | Lunge, Sweep | Ranged, narrow hitboxes |

4. **Map attacks** — update `AssignHitboxIds()` for that character's AttackData assets
5. **Keep it idempotent** — check for existing children/components before creating
