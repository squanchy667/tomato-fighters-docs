# Grok Prompts — Test Scene Environment Art

**Purpose:** Copy-paste prompts for Grok image generation. 6 layers for the dark medieval forest test arena.
**Style Lock:** Flat cel-shaded dark graphic novel, strong ink outlines, no gradients, dark neutrals.

---

## Shared Style Prefix (Prepend to ALL prompts)

```
flat cel-shaded dark graphic novel illustration, strong dark ink outlines,
rough hand-drawn quality, restrained color palette, dark neutrals,
no gradients, no realistic lighting, no soft edges, bold and flat,
medieval fantasy, gothic, atmospheric, 2D game art
```

---

## Layer 1: Background (Distant Sky / Slowest Parallax)

**File:** `bg_forest_distant.png` — **3072 x 1792 px**

```
Wide panoramic background for a 2D side-scrolling beat 'em up game.
Dark medieval mountain range at dusk, viewed from a distance.
Flat cel-shaded illustration style, strong dark ink outlines, dark graphic novel aesthetic, rough hand-drawn quality.

Massive blue-grey mountain silhouettes dominating the entire image, layered from foreground to far distance.
Multiple mountain ridges overlapping — closer mountains darker, distant mountains lighter blue-grey.
Dark navy sky visible only at the very top, narrow strip, no stars, no moon.
Black treeline silhouettes only as a thin strip at the very bottom edge.
Pale cream fog wisps drifting between the mountain layers, filling the valleys.
Mountains are the focus — they fill 70% of the image vertically.

Entire palette is dark neutrals and muted blues only — no bright colors anywhere.
Atmospheric, ominous, still. No characters, no foreground elements, no ground.
Horizontal composition, seamless left-right edges.
No gradients — all color transitions are flat blocks.
Resolution: 3072 x 1792 pixels.
```

**Grok settings:** Landscape aspect ratio. If Grok asks for dimensions, request widest available.

---

## Layer 2: Midground (Trees & Ruins / Medium Parallax)

**File:** `bg_forest_midground.png` — **2560 x 1280 px** — **TRANSPARENT PNG**

```
Midground layer for a 2D side-scrolling beat 'em up game. Transparent background PNG.
Dark medieval forest clearing with scattered ruins.
Flat cel-shaded dark graphic novel style, strong dark ink outlines, rough hand-drawn quality.

Large dark oak trees with thick gnarled trunks positioned at the LEFT and RIGHT edges of the image.
Moss-covered crumbling grey stone ruins scattered between the trees.
Atmospheric pale cream fog patches hovering at ground level between elements.
Twisted roots emerging from the base of trees.

The CENTER of the image is EMPTY — no trees, no ruins, no elements. This is where gameplay happens.
All visual elements are concentrated in the left third and right third only.

Dark brown tree trunks, deep green moss patches, grey weathered stone blocks, pale cream fog.
Restrained palette — dark neutrals only. No bright colors. No sky visible.
Everything on transparent background — only the tree and ruin elements are opaque.
No gradients, flat color blocks only.
Resolution: 2560 x 1280 pixels.
```

**Grok note:** Stress "transparent background" and "center is empty." You may need to remove the background manually with rembg if Grok fills it in.

---

## Layer 3: Foreground (Decorative Edges / Fastest Parallax)

**File:** `bg_forest_foreground.png` — **2560 x 1280 px** — **TRANSPARENT PNG**

```
Foreground decoration layer for a 2D side-scrolling game. Transparent background PNG.
Dark forest ground-level elements only — everything else is transparent.
Flat cel-shaded dark graphic novel style, strong dark ink outlines, rough hand-drawn quality.

Low dark green bushes along the bottom edge of the image.
Twisted dark brown tree roots creeping along the bottom.
Scattered dead leaves in muted orange-brown tones.
Small mossy rocks and pebbles dotted between the roots and bushes.

Elements exist ONLY in the bottom 20% of the image — the top 80% is completely empty/transparent.
Sparse coverage — these are edge decorations, not a solid mass.

Dark green, dark brown, muted burnt orange leaves.
Restrained palette. No bright colors. Moody and atmospheric.
Everything on transparent background.
No gradients, flat color blocks only.
Resolution: 2560 x 1280 pixels.
```

---

## Layer 4: Ground Strip (Play Surface)

**File:** `ground_forest_floor.png` — **2560 x 384 px** — **OPAQUE**

```
Horizontal ground floor strip for a 2D side-scrolling beat 'em up game.
Forest clearing ground — a packed sandy dirt path where characters walk and fight.
Flat cel-shaded dark graphic novel style, strong dark ink outlines, rough hand-drawn quality.

Warm dark sandy brown earth filling the center of the strip.
Darker dirt and mud along the top and bottom edges.
Small grass tufts growing along the top edge and bottom edge — dark muted green, not bright.
Scattered fallen leaves in muted brown and dull orange tones.
Small pebbles and tiny rocks for texture across the surface.
Subtle footpath worn into the center — slightly lighter than surrounding dirt.

Horizontal strip composition — must tile seamlessly left to right.
Dark restrained earthy palette — sandy brown, dark mud brown, muted green grass, dull orange leaves.
No bright greens, no bright colors anywhere. Earthy and grounded.
Opaque — no transparency needed. Fully filled.
No gradients, flat color zones only.
Resolution: 2560 x 384 pixels.
```

---

## Layer 5: Left Wall (Arena Boundary)

**File:** `wall_left_stone.png` — **256 x 1280 px** — **TRANSPARENT PNG**

```
Tall vertical stone wall segment for a 2D side-scrolling game. Transparent background PNG.
Ancient crumbling stone wall — the left boundary of a forest arena.
Flat cel-shaded dark graphic novel style, strong dark ink outlines, rough hand-drawn quality.

Grey weathered stone blocks stacked vertically, with dark mortar lines between them.
Cracks and missing chunks along the top edge — crumbling and ancient.
Dark green moss growing in the crevices and gaps between stones.
Small dark brown vines creeping up from the bottom third.
A few stones jutting out unevenly from the wall face.

The wall fills the frame vertically — tall narrow composition.
Wall shape is on the RIGHT side of the image (this is the left arena boundary, so players see the inner face).
Transparent background outside the wall shape.

Dark restrained palette — grey weathered stone, dark green moss, dark brown vines, black mortar.
No bright colors. Ancient, weathered, atmospheric.
No gradients, flat color blocks only.
Resolution: 256 x 1280 pixels.
```

---

## Layer 6: Right Wall (Arena Boundary)

**File:** `wall_right_stone.png` — **256 x 1280 px** — **TRANSPARENT PNG**

```
Tall vertical stone wall segment for a 2D side-scrolling game. Transparent background PNG.
Ancient crumbling stone wall — the right boundary of a forest arena.
Flat cel-shaded dark graphic novel style, strong dark ink outlines, rough hand-drawn quality.

Grey weathered stone blocks stacked vertically, with dark mortar lines between them.
Cracks and missing chunks along the top edge — crumbling and ancient.
Dark green moss growing in the crevices and gaps between stones.
Small dark brown vines creeping up from the bottom third.
Thicker wall than the left — slightly more intact, with a fallen stone at the base.

The wall fills the frame vertically — tall narrow composition.
Wall shape is on the LEFT side of the image (this is the right arena boundary, so players see the inner face).
Transparent background outside the wall shape.

Dark restrained palette — grey weathered stone, dark green moss, dark brown vines, black mortar.
No bright colors. Ancient, weathered, atmospheric.
No gradients, flat color blocks only.
Resolution: 256 x 1280 pixels.
```

**Alternative:** Generate only the left wall and mirror/flip it horizontally in an image editor for the right wall.

---

## Production Order

1. **Background** first — establishes the palette and mood
2. **Ground strip** — establishes the play surface tone
3. **Midground** — trees and ruins framing the arena (reference background palette)
4. **Foreground** — bottom decorations (reference ground strip palette)
5. **Walls** — left wall, then mirror or generate right variant

## Post-Processing Checklist

- [ ] Remove backgrounds on layers 2, 3, 5, 6 (use rembg if Grok doesn't generate transparent)
- [ ] Verify dimensions match spec (resize if needed, maintain 128 PPU scale)
- [ ] Check palette consistency across all 6 layers — dark neutrals, no bright outliers
- [ ] Place files in `Assets/Art/Environment/TestArena/`

## Grok Tips

- Generate 2-3 variants per layer, pick the most consistent one
- If possible, generate all layers in the same session to maintain style coherence
- For transparent PNGs: Grok may not support transparency natively — generate on white or solid color background, then remove with rembg
- If Grok struggles with exact pixel dimensions, generate at the closest aspect ratio and resize in an editor
