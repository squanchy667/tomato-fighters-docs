# Test Scene — Full Art Design Specification

**Version:** 1.0.0
**Purpose:** Art asset spec for the T013 test scene. Covers environment, 6 enemies, and UI.
**Art Tool:** AI generation (refine output manually as needed)

---

## 1. Art Style Guide

### Style DNA (Locked — Shared With Player Characters)

| Element | Specification |
|---------|--------------|
| **Rendering** | Flat cel-shaded, dark graphic novel illustration |
| **Line art** | Strong confident dark ink outlines, slightly rough/hand-drawn quality |
| **Coloring** | Flat cel-shading — minimal gradient, large clean color zones |
| **Palette** | Restrained — one dominant color per character against dark neutrals (black, dark grey, dark brown) |
| **Lighting** | Flat — no realistic light sources. Dark/light zones as flat color blocks, not gradients |
| **Edges** | No soft edges. Everything bold, flat, outlined |
| **Mood** | Comedic premise, serious execution. Environments genuinely atmospheric and slightly dangerous |
| **World feel** | Medieval-fantasy with gothic and arcane elements. Stone, wood, moss, candlelight, ruins, fog |

### Style Reference Keywords (For AI Prompts)
```
flat cel-shaded dark graphic novel illustration, strong dark ink outlines,
rough hand-drawn quality, restrained color palette, dark neutrals,
no gradients, no realistic lighting, no soft edges, bold and flat,
medieval fantasy, gothic, atmospheric
```

---

## 2. Technical Specifications

### Sprite Sheet Format (Enemy Characters)

All enemy sprite sheets must match the existing player character pipeline:

| Property | Value | Notes |
|----------|-------|-------|
| **Frame size** | 128 x 192 px | Width x Height per frame |
| **PPU** | 128 | Pixels Per Unit — 1 frame = 1x1.5 Unity units |
| **Pivot** | [0.5, 0.0] | Bottom-center |
| **Grid (locomotion)** | 4 cols x 4 rows | Up to 16 frames per sheet |
| **Grid (actions)** | 4 cols x 2 rows | Up to 8 frames per sheet |
| **Format** | PNG with transparency | Background removed |
| **Filter mode** | Bilinear | As set in metadata.json |
| **Compression** | Uncompressed | Pipeline default |

### Environment Art Dimensions

Camera orthographic size = 7, arena = 20x10 Unity units.

| Layer | Dimensions (px) | Unity Size | Notes |
|-------|-----------------|------------|-------|
| **Background** (sky/distant) | 3072 x 1792 | 24 x 14 units | Extends beyond arena for parallax margin |
| **Midground** (trees/ruins) | 2560 x 1280 | 20 x 10 units | Matches arena footprint |
| **Foreground** (bushes/roots) | 2560 x 1280 | 20 x 10 units | Mostly transparent, edge decorations |
| **Ground strip** | 2560 x 384 | 20 x 3 units | Sandy forest floor, placed at bottom of arena |
| **Wall visuals** (L/R) | 256 x 1280 | 2 x 10 units | Stone/wood barrier with moss |

All environment art at **128 PPU** to match character scale.

### File Locations

```
Assets/animations/
├── tomato_berserker_animations/
│   ├── metadata.json
│   └── Sprites/
│       ├── tomato_berserker_idle.png
│       ├── tomato_berserker_attack.png
│       ├── tomato_berserker_hurt.png
│       └── tomato_berserker_death.png
├── corn_knight_animations/
│   ├── metadata.json
│   └── Sprites/
│       ├── corn_knight_idle.png
│       ├── corn_knight_attack.png
│       ├── corn_knight_hurt.png
│       └── corn_knight_death.png
├── onion_knight_animations/
│   ├── ...
├── garlic_vampire_animations/
│   ├── ...
├── eggplant_wizard_animations/
│   ├── ...
└── mushroom_ghost_animations/
    ├── ...

Assets/Art/Environment/TestArena/
├── bg_forest_distant.png        (3072x1792)
├── bg_forest_midground.png      (2560x1280)
├── bg_forest_foreground.png     (2560x1280)
├── ground_forest_floor.png      (2560x384)
├── wall_left_stone.png          (256x1280)
└── wall_right_stone.png         (256x1280)
```

---

## 3. Environment Art Briefs

### 3a. Background Layer (Distant — Slowest Parallax)

**What:** Dark forest skyline with distant mountains/treeline fading into mist.

**Palette:** Dark navy sky, muted blue-grey mountains, silhouetted black treeline, wisps of pale fog.

**AI Prompt:**
```
Wide panoramic background for a 2D side-scrolling game.
Dark medieval forest at dusk. Flat cel-shaded illustration style,
strong ink outlines, dark graphic novel aesthetic.
Dark navy sky with no stars, distant blue-grey mountain silhouettes,
black treeline in middle distance, pale fog wisps drifting between layers.
No bright colors — entire palette is dark neutrals and muted blues.
Atmospheric and slightly ominous. No characters, no foreground elements.
Horizontal composition, seamless left-right edges if possible.
Resolution: 3072 x 1792 pixels. Transparent or dark navy background.
```

### 3b. Midground Layer (Trees/Ruins — Medium Parallax)

**What:** Large dark trees, moss-covered stone ruins, atmospheric fog patches. Frames the play area but doesn't obstruct it.

**Palette:** Dark brown tree trunks, deep green moss, grey stone ruins, pale fog. Elements mostly at the edges (left/right), center relatively open.

**AI Prompt:**
```
Midground layer for a 2D side-scrolling beat 'em up.
Dark medieval forest clearing. Flat cel-shaded dark graphic novel style,
strong ink outlines, rough hand-drawn quality.
Large dark oak trees with thick trunks at left and right edges,
moss-covered crumbling stone ruins scattered between trees,
atmospheric pale fog patches at ground level.
Center area is OPEN (this is where gameplay happens).
Elements concentrated at left and right thirds of the image.
Dark brown trunks, deep green moss, grey weathered stone, pale cream fog.
Restrained palette — dark neutrals only. No bright colors.
Medieval fantasy atmosphere. Resolution: 2560 x 1280 pixels.
Transparent background (PNG) — only the tree/ruin elements are opaque.
```

### 3c. Foreground Layer (Decorative Edges — Fastest Parallax)

**What:** Low bushes, twisted roots, fallen leaves, small rocks at the very bottom edges. Partially overlaps the play area for depth.

**Palette:** Dark green bushes, brown roots/branches, scattered autumn leaves (muted orange/brown).

**AI Prompt:**
```
Foreground decoration layer for a 2D side-scrolling game.
Dark forest ground-level elements. Flat cel-shaded dark graphic novel style,
strong ink outlines, rough hand-drawn quality.
Low dark bushes and twisted tree roots along the bottom edge,
scattered dead leaves and small mossy rocks.
Elements ONLY at the bottom 20% of the image — rest is transparent.
Dark green, dark brown, muted orange-brown leaves.
Restrained palette. No bright colors. Atmospheric and moody.
Resolution: 2560 x 1280 pixels. Transparent background (PNG).
```

### 3d. Ground Strip (Play Surface)

**What:** The actual ground the characters walk on. Sandy forest floor with dirt path, grass edges, leaf litter.

**Palette:** Warm dark sand/earth tones, grass tufts at edges, scattered leaves.

**AI Prompt:**
```
Ground floor tile for a 2D side-scrolling beat 'em up.
Forest clearing ground — packed sandy dirt path.
Flat cel-shaded dark graphic novel style, strong ink outlines.
Warm dark earth tones — sandy brown center, darker dirt edges,
small grass tufts growing along the top and bottom edges,
scattered fallen leaves and small pebbles for texture.
Horizontal strip composition — seamless left-right tiling preferred.
Dark restrained palette. No bright greens — muted and earthy.
Resolution: 2560 x 384 pixels. Opaque (no transparency needed).
```

### 3e. Wall Visuals (Left + Right Arena Boundaries)

**What:** Ancient stone walls with moss and cracks, replacing the invisible colliders. Should look like crumbling ruins that naturally wall off the arena.

**AI Prompt (Left Wall):**
```
Tall stone wall segment for a 2D side-scrolling game.
Ancient crumbling stone wall, left side of a forest arena.
Flat cel-shaded dark graphic novel style, strong ink outlines.
Grey weathered stone blocks with dark mortar lines,
cracks and chunks missing from top edge,
dark green moss growing in crevices,
small vines creeping up from bottom.
Vertical composition — wall fills the frame.
Dark restrained palette — grey stone, dark green moss, dark brown vines.
Resolution: 256 x 1280 pixels. Transparent background outside the wall shape.
```

*(Mirror/flip for right wall, or generate a distinct variant.)*

---

## 4. Enemy Art Briefs

### Shared Enemy Style Rules

- Same flat cel-shaded dark graphic novel style as player characters
- Strong dark ink outlines, rough hand-drawn quality
- One dominant color per enemy against dark neutrals
- Medieval armor/weapons — scavenged, battle-worn, cracked
- Goofy-scary tone: the designs are inherently comedic (vegetables in armor) but rendered seriously
- Pure white background for generation, then remove background for sprite sheets
- Side-view or 3/4 view facing LEFT (standard for 2D side-scrollers; flip in-engine for right-facing)

### Animations Required Per Enemy

| Animation | Grid | Frames | Loop | Purpose |
|-----------|------|--------|------|---------|
| **idle** | 4x4 | 12-17 | Yes | Breathing, slight movement, personality idle |
| **walk** | 4x4 | 8-12 | Yes | Walking toward player (patrol/chase) |
| **attack** | 4x2 | 6-8 | No | Primary attack with telegraph wind-up |
| **hurt** | 4x2 | 4-6 | No | Hit reaction / stagger |
| **death** | 4x2 | 6-8 | No | Death collapse |
| **stun** | 4x2 | 4-6 | Yes | Stunned/dazed state (stars, wobble) |

**Total per enemy: 6 sprite sheets, 1 metadata.json**

---

### Enemy 1 — TOMATO BERSERKER

**Test Scene Tier:** Fighter (mid-tier, baseline enemy)

| Property | Value |
|----------|-------|
| **Dominant color** | Saturated red |
| **Accent colors** | Dark iron gray, leafy green |
| **Size** | Short and round — knee-height of player (~0.65x1.0 scale) |
| **Shape language** | Circular, compact, low to ground |
| **Weapon** | Small iron cleaver or spiked fist |
| **Personality** | Furious, chaotic, charges in recklessly |
| **Telegraph type** | Normal (can be deflected/clashed) |

**AI Prompt (Concept Art / Idle Base):**
```
Full-body character concept art in dark graphic novel illustration style.
Flat cel-shaded colors, strong dark ink outlines, rough hand-drawn quality.
Restrained palette — saturated red dominant color against dark neutrals.
Pure white background, no floor, no shadow.

Character: a small round tomato creature wearing medieval battle armor.
Short and compact — approximately knee-height of a human.
Bright saturated red tomato body, spherical and plump,
small angry eyes with heavy brow, gritted teeth showing,
a tiny green leaf stem on top like a helmet crest.
Wearing dark iron gray chest plate, cracked and dented,
small pauldrons on stubby arms, iron gauntlets gripping a small cleaver.
Short stumpy legs in dark iron greaves.
All armor scuffed, dented, battle-worn.

Pose: aggressive forward lean, cleaver raised, ready to charge.
Small body radiating fury. Weight low and forward.

Style: flat cel-shaded, bold ink outlines, no gradients,
saturated red and dark iron gray palette.
Goofy concept but rendered with serious dark graphic novel quality.
```

**Idle Animation Notes:** Angry breathing — body inflates/deflates slightly. Cleaver twitches. Occasional foot stomp.

**Attack Animation Notes:** Wind-up: pulls cleaver back, leans further forward. Strike: lunges forward with overhead chop. Short range, fast recovery.

---

### Enemy 2 — CORN KNIGHT

**Test Scene Tier:** Heavy (slow, powerful)

| Property | Value |
|----------|-------|
| **Dominant color** | Golden yellow |
| **Accent colors** | Polished silver, deep green husk, royal red sash |
| **Size** | Tall and cylindrical — roughly human height (~0.9x1.35 scale) |
| **Shape language** | Vertical, rigid, upright |
| **Weapon** | Long lance or halberd |
| **Personality** | Noble, stiff, takes offense at everything |
| **Telegraph type** | Unstoppable (must dodge, cannot deflect/clash) |

**AI Prompt (Concept Art / Idle Base):**
```
Full-body character concept art in dark graphic novel illustration style.
Flat cel-shaded colors, strong dark ink outlines, rough hand-drawn quality.
Restrained palette — golden yellow dominant color against dark neutrals.
Pure white background, no floor, no shadow.

Character: a tall corn cob creature in full medieval knight armor.
Tall and cylindrical body — roughly human height but narrow.
Golden yellow corn kernels visible through gaps in armor,
green husk leaves forming a cape flowing behind,
stern disapproving face with small eyes and a thin frown.
Wearing polished silver full plate armor with golden inlay,
a royal red sash across the chest plate,
tall plumed helmet (husk leaves as plume).
Holding a long silver lance upright in gauntleted hand.

Pose: standing at rigid attention, lance held vertically,
chin raised in noble disdain. Perfectly upright posture.
Weight centered, immovable, looking down at the viewer.

Style: flat cel-shaded, bold ink outlines, no gradients,
golden yellow and polished silver palette with dark neutrals.
Goofy concept but rendered as a genuinely imposing knight.
```

**Idle Animation Notes:** Minimal movement — slight sway, cape flutter. Occasionally adjusts grip on lance with visible irritation.

**Attack Animation Notes:** Slow wind-up: raises lance high, red telegraph flash. Powerful downward thrust. Long telegraph duration (1.2s). Unstoppable — player must dodge.

---

### Enemy 3 — ONION CRYBABY KNIGHT

**Test Scene Tier:** Scrapper (weak, fast attacks)

| Property | Value |
|----------|-------|
| **Dominant color** | Pale lavender-white |
| **Accent colors** | Dark iron gray, muted silver, faded cape brown |
| **Size** | Short and round — similar to Tomato but slightly taller (~0.7x1.1 scale) |
| **Shape language** | Round, slumped, heavy |
| **Weapon** | Small dented shield + short sword |
| **Personality** | Perpetually sad, fights through tears, surprisingly persistent |
| **Telegraph type** | Normal (can be deflected/clashed) |

**AI Prompt (Concept Art / Idle Base):**
```
Full-body character concept art in dark graphic novel illustration style.
Flat cel-shaded colors, strong dark ink outlines, rough hand-drawn quality.
Restrained palette — pale lavender-white dominant color against dark neutrals.
Pure white background, no floor, no shadow.

Character: a round onion creature in dented medieval armor.
Short and round body — slightly taller than knee-height.
Pale lavender-white onion layers visible at the top,
large watery eyes constantly streaming tears down its face,
quivering lower lip, despondent expression.
Wearing dark iron gray dented armor, too big for its body,
a faded brown tattered cape dragging on the ground,
small muted silver shield held loosely in one hand,
short chipped sword in the other.
Armor scratched and badly maintained.

Pose: slumped shoulders, head slightly bowed,
weapons held loosely at sides as if too tired to raise them.
Tears visibly dripping. Weight sagging downward.

Style: flat cel-shaded, bold ink outlines, no gradients,
pale lavender-white and dark iron gray palette.
Pathetic and sad but rendered with genuine pathos, not mockery.
```

**Idle Animation Notes:** Constant crying — tear drops falling, body shuddering with sobs. Occasionally wipes eye with shield arm.

**Attack Animation Notes:** Reluctant swing — winds up slowly while crying harder, then flails forward with sword. Fast but weak.

---

### Enemy 4 — GARLIC VAMPIRE

**Test Scene Tier:** Bruiser (strongest, most dangerous)

| Property | Value |
|----------|-------|
| **Dominant color** | Deep crimson (seven capes dominate silhouette) |
| **Accent colors** | Off-white cream, Victorian black, gold bat medallion |
| **Size** | Small body but enormous cape silhouette (~1.0x1.5 scale with capes) |
| **Shape language** | Wide, layered, dramatic cape spread |
| **Weapon** | Cape sweep attack + bite |
| **Personality** | Dramatic, theatrical, thinks it's terrifying |
| **Telegraph type** | Unstoppable (must dodge) |

**AI Prompt (Concept Art / Idle Base):**
```
Full-body character concept art in dark graphic novel illustration style.
Flat cel-shaded colors, strong dark ink outlines, rough hand-drawn quality.
Restrained palette — deep crimson dominant color against dark neutrals.
Pure white background, no floor, no shadow.

Character: a small round garlic bulb creature as a vampire lord.
Small off-white cream colored garlic body at center,
but SEVEN dramatic deep crimson capes layered around it,
spreading wide to create an enormous imposing silhouette.
Tiny glowing red eyes peering from between cape layers,
two small fangs visible in a smug grin,
a gold bat-shaped medallion clasping the innermost cape.
Victorian black collar visible at the neck,
small clawed feet barely visible beneath the cape mass.

Pose: capes dramatically spread wide to maximum width,
head tilted up in theatrical villain pose,
one small clawed hand extended from cape folds.
The capes ARE the character — they dominate the entire silhouette.

Style: flat cel-shaded, bold ink outlines, no gradients,
deep crimson and Victorian black palette with gold accent.
Absurd concept rendered as a genuinely menacing gothic figure.
```

**Idle Animation Notes:** Capes billow and shift constantly. Occasionally spreads them wider for dramatic effect, then settles back. Eyes glow pulse.

**Attack Animation Notes:** Telegraph: capes pull inward (gathering), red flash. Strike: explosive cape sweep across entire hitbox width (2.4 units). Hits both sides. Unstoppable.

---

### Enemy 5 — EGGPLANT WIZARD

**Test Scene Tier:** (6th slot — ranged/special)

| Property | Value |
|----------|-------|
| **Dominant color** | Deep rich purple |
| **Accent colors** | Dark neutral robes, small purple arcane sparks, wood-brown staff |
| **Size** | Tall and slender — slightly above human height (~0.8x1.35 scale) |
| **Shape language** | Tall teardrop, elegant, vertical with wide hat brim |
| **Weapon** | Gnarled wooden staff with purple crystal tip |
| **Personality** | Aloof, condescending, casts spells carelessly |
| **Telegraph type** | Normal (can be deflected/clashed) |

**AI Prompt (Concept Art / Idle Base):**
```
Full-body character concept art in dark graphic novel illustration style.
Flat cel-shaded colors, strong dark ink outlines, rough hand-drawn quality.
Restrained palette — deep rich purple dominant color against dark neutrals.
Pure white background, no floor, no shadow.

Character: a tall slender eggplant creature as an arcane wizard.
Deep rich purple eggplant body — teardrop shaped, narrow at bottom.
Wearing dark neutral hooded robes with frayed edges,
a wide-brimmed wizard hat that extends the eggplant's natural shape,
half-lidded contemptuous eyes, thin disapproving mouth.
Carrying a gnarled wood-brown staff with a small purple crystal at the tip,
small purple arcane sparks drifting lazily around the crystal.
Long thin arms emerging from oversized robe sleeves.
Small curled shoes barely visible under robe hem.

Pose: leaning slightly on staff with one hand,
other hand raised dismissively as if shooing away something beneath notice.
Elegant, vertical posture with slight condescending lean.

Style: flat cel-shaded, bold ink outlines, no gradients,
deep purple and dark neutral palette with tiny purple spark accents.
Rendered as a genuinely mysterious and powerful-looking wizard.
```

**Idle Animation Notes:** Floating slightly (robes drift). Purple sparks orbit lazily. Occasionally yawns or examines fingernails.

**Attack Animation Notes:** Staff raised, purple energy gathers (telegraph). Releases arcane bolt forward. Can be clashed.

---

### Enemy 6 — MUSHROOM GHOST

**Test Scene Tier:** Weakling (weakest, easy to defeat)

| Property | Value |
|----------|-------|
| **Dominant color** | Muted dusty brown (cap) and pale cream (body) |
| **Accent colors** | Dark iron pauldron gray, chain black, dark smoke wisp |
| **Size** | Short and wide-capped — stocky, ground-level (~0.65x1.0 scale) |
| **Shape language** | Wide dome on top, stubby stem body, small dangling legs |
| **Weapon** | Ghostly smoke puff / headbutt |
| **Personality** | Eerily silent, just... watches. Then attacks. |
| **Telegraph type** | Normal (can be deflected/clashed) |

**AI Prompt (Concept Art / Idle Base):**
```
Full-body character concept art in dark graphic novel illustration style.
Flat cel-shaded colors, strong dark ink outlines, rough hand-drawn quality.
Restrained palette — muted dusty brown dominant against dark neutrals.
Pure white background, no floor, no shadow.

Character: a short stocky mushroom creature with ghostly properties.
Wide muted dusty brown mushroom cap dominating the top half,
pale cream stubby stem body beneath,
empty hollow black eyes with tiny pale pupils,
no visible mouth — just smooth pale surface.
Wearing a single dark iron gray pauldron on one shoulder,
a thin black chain draped across the body,
dark smoke wisps trailing from beneath the cap edges.
Small stubby legs dangling beneath the stem,
barely touching the ground — hovering slightly.

Pose: perfectly still and centered, facing forward,
slight tilt of the cap as if observing.
Smoke wisps drifting upward from edges.
Unsettling stillness — the lack of expression IS the expression.

Style: flat cel-shaded, bold ink outlines, no gradients,
dusty brown and pale cream palette with dark smoke accents.
The quietest and most genuinely unsettling enemy in the roster.
```

**Idle Animation Notes:** Almost no movement. Slight hover bob. Smoke wisps drift. Occasionally the cap tilts slightly — that's it. Unnerving stillness.

**Attack Animation Notes:** Short wind-up: lurches forward suddenly (breaking the stillness). Headbutt/smoke puff. Quick, weak, surprising.

---

## 5. Enemy-to-Tier Mapping (Test Scene Creator)

Maps the 6 enemies to the test scene's combat tier system:

| Tier | Enemy | HP | Pressure | Attack | Telegraph | Knockback |
|------|-------|-----|----------|--------|-----------|-----------|
| **Bruiser** | Garlic Vampire | 150 | 70 | 2.0x, both sides | Unstoppable, 1.5s | (8, 2) |
| **Heavy** | Corn Knight | 130 | 60 | 1.5x | Unstoppable, 1.2s | (5, 1) |
| **Fighter** | Tomato Berserker | 100 | 50 | 1.0x | Normal, 1.0s | (3, 0) |
| **Caster** | Eggplant Wizard | 80 | 40 | 0.8x | Normal, 0.8s | (2, 0) |
| **Scrapper** | Onion Crybaby | 70 | 35 | 0.6x | Normal, 0.7s | (2, 0) |
| **Weakling** | Mushroom Ghost | 50 | 25 | 0.3x | Normal, 0.5s | (1, 0) |

### Test Scene Positions

```
         Garlic Vampire (Bruiser)
              (0, 2.5)

  Tomato (Fighter)          Onion (Scrapper)
    (3, 1)                    (7, 1.5)

              Corn Knight (Heavy)
Player          (5, -1.5)
(-3, 0)
                        Eggplant (Caster)
                          (6, 0)

              Mushroom Ghost (Weakling)
                (7, -2.5)
```

---

## 6. UI Art Briefs

### Health Bars

| Element | Spec |
|---------|------|
| **Frame** | Dark iron gray border, 2px thick ink outline, slightly rough edges |
| **Player fill** | Muted green gradient → flat green (no gradient in cel-shaded style, use two-tone: bright green + dark green shadow) |
| **Enemy fill** | Saturated red, same two-tone approach |
| **Boss fill** | Deep gold with dark gold shadow |
| **Background** | Near-black (0.1, 0.1, 0.1) |
| **Size** | Player: 200x20px. Enemy: 80x12px (world-space, above head) |

### Defense Event Popups

Style: Bold hand-drawn text, slight rotation for impact feel.

| Event | Color | Text |
|-------|-------|------|
| Deflect | Bright cyan | DEFLECTED! |
| Clash | Golden yellow | CLASHED! |
| Dodge | Pale white | DODGED! |
| Stun | Red-orange | STUNNED! |
| Punish | Deep crimson | PUNISH! |

### Combo Counter

- Large bold number, dark ink outline
- Color shifts: white (1-5), yellow (6-10), orange (11-20), red (21+)
- Slight shake/pulse on increment

---

## 7. metadata.json Templates

### Example: Tomato Berserker

```json
{
  "character_name": "tomato_berserker",
  "generated_at": "",
  "generated_by": "AI Generation + Manual Refinement",
  "bg_removal_method": "rembg + green_defringe + smoke_removal",
  "game_profile": {
    "game_type": "2d_roguelite",
    "art_style": "ai_generated_hd",
    "export_target": "unity",
    "filter_mode": "bilinear"
  },
  "animations": {
    "idle": {
      "spritesheet": "Sprites/tomato_berserker_idle.png",
      "frame_w": 128,
      "frame_h": 192,
      "cols": 4,
      "rows": 4,
      "n_frames": 12,
      "fps": 12,
      "loop": true,
      "pivot": [0.5, 0.0],
      "ppu": 128
    },
    "walk": {
      "spritesheet": "Sprites/tomato_berserker_walk.png",
      "frame_w": 128,
      "frame_h": 192,
      "cols": 4,
      "rows": 4,
      "n_frames": 8,
      "fps": 12,
      "loop": true,
      "pivot": [0.5, 0.0],
      "ppu": 128
    },
    "attack": {
      "spritesheet": "Sprites/tomato_berserker_attack.png",
      "frame_w": 128,
      "frame_h": 192,
      "cols": 4,
      "rows": 2,
      "n_frames": 7,
      "fps": 12,
      "loop": false,
      "pivot": [0.5, 0.0],
      "ppu": 128
    },
    "hurt": {
      "spritesheet": "Sprites/tomato_berserker_hurt.png",
      "frame_w": 128,
      "frame_h": 192,
      "cols": 4,
      "rows": 2,
      "n_frames": 4,
      "fps": 12,
      "loop": false,
      "pivot": [0.5, 0.0],
      "ppu": 128
    },
    "death": {
      "spritesheet": "Sprites/tomato_berserker_death.png",
      "frame_w": 128,
      "frame_h": 192,
      "cols": 4,
      "rows": 2,
      "n_frames": 7,
      "fps": 10,
      "loop": false,
      "pivot": [0.5, 0.0],
      "ppu": 128
    },
    "stun": {
      "spritesheet": "Sprites/tomato_berserker_stun.png",
      "frame_w": 128,
      "frame_h": 192,
      "cols": 4,
      "rows": 2,
      "n_frames": 4,
      "fps": 8,
      "loop": true,
      "pivot": [0.5, 0.0],
      "ppu": 128
    }
  }
}
```

*(Duplicate this template for all 6 enemies, changing `character_name` and frame counts as appropriate.)*

---

## 8. Art Production Workflow

### Step 1: Generate Concept Art
1. Use the AI prompts in Section 4 to generate concept art for each enemy
2. Generate multiple variants, pick the best
3. Ensure consistent style across all 6 enemies (same session/seed if possible)

### Step 2: Generate Animation Frames
For each enemy, generate the 6 animation states:
1. **Idle:** 12-17 frames of subtle breathing/movement
2. **Walk:** 8-12 frames of locomotion cycle
3. **Attack:** 6-8 frames (wind-up → strike → recovery)
4. **Hurt:** 4-6 frames (impact → stagger → recover)
5. **Death:** 6-8 frames (collapse → settle)
6. **Stun:** 4-6 frames (dazed wobble, looping)

### Step 3: Assemble Sprite Sheets
1. Each frame must be exactly **128x192px**
2. Arrange in grid: left-to-right, top-to-bottom
3. Locomotion: 4 cols x 4 rows (512x768px sheet)
4. Actions: 4 cols x 2 rows (512x384px sheet)
5. Remove background (rembg or manual)
6. Export as PNG with transparency

### Step 4: Create metadata.json
1. Copy the template from Section 7
2. Adjust `n_frames` to match actual frame count per animation
3. Place in `Assets/animations/{enemy_name}_animations/metadata.json`

### Step 5: Generate Environment Art
1. Use the prompts in Section 3 for each layer
2. Ensure consistent palette across all layers (dark neutrals + accent pops)
3. Export at specified dimensions with transparency where noted
4. Place in `Assets/Art/Environment/TestArena/`

### Step 6: Import Into Unity Pipeline
1. Run **TomatoFighters > Import Sprite Sheets > All Characters** (after adding enemy importers)
2. Run **TomatoFighters > Build Animations > All Characters**
3. Run **TomatoFighters > Stamp Animation Events**
4. Update `MovementTestSceneCreator.cs` to reference new enemy prefabs and environment art

---

## 9. Open Questions

1. **Player character art updates:** Do the existing 4 player characters need art updates to match this style, or are they already in the correct style?
2. **VFX:** Should we create hit spark, telegraph indicator, and stun star VFX sprites for this pass, or defer to a later task?
3. **Animation tool:** Will you use frame-by-frame AI generation, or an AI animation tool that generates full sequences from a single concept?
4. **Enemy SpriteSheetImporter:** The current importer only handles the 4 player characters. We'll need to extend it (or create an enemy variant) to import the 6 enemy sprite sheets. This is a code task for T013.

---

## 10. Asset Checklist

### Environment (6 assets)
- [ ] `bg_forest_distant.png` (3072x1792)
- [ ] `bg_forest_midground.png` (2560x1280)
- [ ] `bg_forest_foreground.png` (2560x1280)
- [ ] `ground_forest_floor.png` (2560x384)
- [ ] `wall_left_stone.png` (256x1280)
- [ ] `wall_right_stone.png` (256x1280)

### Tomato Berserker (6 sheets + metadata)
- [ ] `tomato_berserker_idle.png` (512x768)
- [ ] `tomato_berserker_walk.png` (512x768)
- [ ] `tomato_berserker_attack.png` (512x384)
- [ ] `tomato_berserker_hurt.png` (512x384)
- [ ] `tomato_berserker_death.png` (512x384)
- [ ] `tomato_berserker_stun.png` (512x384)
- [ ] `metadata.json`

### Corn Knight (6 sheets + metadata)
- [ ] `corn_knight_idle.png` (512x768)
- [ ] `corn_knight_walk.png` (512x768)
- [ ] `corn_knight_attack.png` (512x384)
- [ ] `corn_knight_hurt.png` (512x384)
- [ ] `corn_knight_death.png` (512x384)
- [ ] `corn_knight_stun.png` (512x384)
- [ ] `metadata.json`

### Onion Crybaby Knight (6 sheets + metadata)
- [ ] `onion_knight_idle.png` (512x768)
- [ ] `onion_knight_walk.png` (512x768)
- [ ] `onion_knight_attack.png` (512x384)
- [ ] `onion_knight_hurt.png` (512x384)
- [ ] `onion_knight_death.png` (512x384)
- [ ] `onion_knight_stun.png` (512x384)
- [ ] `metadata.json`

### Garlic Vampire (6 sheets + metadata)
- [ ] `garlic_vampire_idle.png` (512x768)
- [ ] `garlic_vampire_walk.png` (512x768)
- [ ] `garlic_vampire_attack.png` (512x384)
- [ ] `garlic_vampire_hurt.png` (512x384)
- [ ] `garlic_vampire_death.png` (512x384)
- [ ] `garlic_vampire_stun.png` (512x384)
- [ ] `metadata.json`

### Eggplant Wizard (6 sheets + metadata)
- [ ] `eggplant_wizard_idle.png` (512x768)
- [ ] `eggplant_wizard_walk.png` (512x768)
- [ ] `eggplant_wizard_attack.png` (512x384)
- [ ] `eggplant_wizard_hurt.png` (512x384)
- [ ] `eggplant_wizard_death.png` (512x384)
- [ ] `eggplant_wizard_stun.png` (512x384)
- [ ] `metadata.json`

### Mushroom Ghost (6 sheets + metadata)
- [ ] `mushroom_ghost_idle.png` (512x768)
- [ ] `mushroom_ghost_walk.png` (512x768)
- [ ] `mushroom_ghost_attack.png` (512x384)
- [ ] `mushroom_ghost_hurt.png` (512x384)
- [ ] `mushroom_ghost_death.png` (512x384)
- [ ] `mushroom_ghost_stun.png` (512x384)
- [ ] `metadata.json`

### UI Elements
- [ ] Health bar frame sprite
- [ ] Defense popup font/style reference

**Total: 43 image assets + 6 metadata files + 6 environment images = 55 files**
