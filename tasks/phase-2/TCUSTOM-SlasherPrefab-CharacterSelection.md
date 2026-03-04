# TCUSTOM: Slasher Prefab + Character Selection in Movement Test Scene

**Pillar:** Combat (Dev 1)
**Phase:** 2
**Priority:** High
**Dependencies:** T011 (combo system), T016 (defense system)

---

## Summary

Add a Slasher character prefab and modify the Movement Test scene creator to support choosing which character to spawn via an editor dropdown window.

## Deliverables

1. `SlasherCharacterCreator.cs` — editor script that builds `Slasher.prefab`
2. `CharacterSelectionWindow.cs` — small `EditorWindow` with `CharacterType` dropdown + "Create Scene" button
3. Updated `MovementTestSceneCreator.cs` — accepts a `CharacterType`, loads the matching prefab
4. Updated `MysticaCharacterCreator.cs` — saves to `Mystica.prefab` instead of `Player.prefab`

---

## Design Decisions

### DD-1: Slasher MovementConfig values

Slasher is the fastest melee character (SPD 1.3). Config tuned for snappy, aggressive feel:

```
moveSpeed = 10
depthSpeed = 6
groundAcceleration = 70
airAcceleration = 40
jumpForce = 14
jumpGravity = 38
coyoteTime = 0.1
jumpBufferTime = 0.12
dashSpeed = 22
dashDuration = 0.12
dashCooldown = 0.45
dashHasIFrames = true
runSpeedMultiplier = 1.6
```

**Rationale:** Higher move/dash speed and acceleration than Mystica (8/20/65) to match Slasher's 1.3 SPD vs Mystica's 1.0. Shorter dash duration with lower cooldown for rapid repositioning.

### DD-2: Slasher ComboDefinition — 6 steps

Per CHARACTER-ARCHETYPES: "4-hit fast slash combo with spinning finisher" + "Lunging thrust" heavy.

| Step | Index | AttackType | Trigger | DmgMult | HitboxId | DashCancel | JumpCancel | Finisher |
|------|-------|------------|---------|---------|----------|------------|------------|----------|
| Slash1 | 0 | Light | Light1 | 1.0 | Slash | no | no | no |
| Slash2 | 1 | Light | Light2 | 1.1 | Slash | yes | no | no |
| Slash3 | 2 | Light | Light3 | 1.2 | Slash | yes | yes | no |
| SpinFinisher | 3 | Light | LightFinisher | 1.6 | SpinSlash | no | no | yes |
| LungeThrust | 4 | Heavy | Heavy1 | 2.0 | Thrust | yes | yes | no |
| PowerThrust | 5 | Heavy | HeavyFinisher | 2.8 | Thrust | no | no | yes |

- `rootLightIndex = 0`, `rootHeavyIndex = 4`
- `defaultComboWindow = 0.4` (slightly shorter than Mystica's 0.5 — faster chains)
- Step 0→1→2→3 light chain, step 4→5 heavy chain

**Rationale:** 4-hit light chain matches the spec. Damage multipliers ramp faster than Mystica's to reflect Slasher's 2.0 ATK identity. Cancels on mid-chain hits reward aggression.

### DD-3: Slasher Hitbox definitions

Three hitbox types — "fast, tight" per the spec:

```csharp
// Slash — tight horizontal box for rapid slashes
hitboxId = "Slash", shape = Box, boxSize = (1.0, 0.3), offset = (0.6, 0.1)

// SpinSlash — circle for the spinning finisher
hitboxId = "SpinSlash", shape = Circle, circleRadius = 0.7, offset = (0.3, 0.1)

// Thrust — narrow, long box for the lunging thrust
hitboxId = "Thrust", shape = Box, boxSize = (1.8, 0.25), offset = (1.0, 0.1)
```

**Rationale:** Slash is narrower than Mystica's Burst (r=0.5 circle) — tight but fast. SpinSlash covers a wide area as the combo payoff. Thrust is the longest hitbox in the game — matches "covers distance" from the spec.

### DD-4: Prefab path convention

Each character gets its own prefab: `Assets/Prefabs/Player/{CharacterName}.prefab`

- `Mystica.prefab` (renamed from `Player.prefab`)
- `Slasher.prefab` (new)

**Rationale:** Clear naming, same folder, scales to Brutor/Viper later. No backwards compatibility needed.

### DD-5: Character selection via EditorWindow

Replace the direct `Create Movement Test Scene` menu item with a small `EditorWindow`:

```
┌─ Movement Test Scene ──────────────────┐
│                                        │
│  Character:  [Slasher ▼]               │
│                                        │
│        [ Create Scene ]                │
└────────────────────────────────────────┘
```

- Menu: `TomatoFighters > Create Movement Test Scene` opens the window
- Dropdown lists all `CharacterType` enum values
- Button calls `MovementTestSceneCreator.CreateTestScene(selectedType)`
- Prefab path resolved as `Assets/Prefabs/Player/{characterType}.prefab`
- If prefab doesn't exist, logs error and does NOT fall back — tells user to run the character creator

**Rationale:** Simple, zero-ceremony, scales to 4 characters. No backwards compatibility — old menu item replaced entirely.

### DD-6: Shared animator controller

Slasher will use the same `TomatoFighter_Controller.controller` as Mystica for now. No character-specific animations exist yet. The combo system works via timer fallback (`useTimerFallback=true`), so missing animation events are not a blocker.

### DD-7: AttackData SOs — created as stubs

The creator will attempt to load AttackData SOs from `Assets/ScriptableObjects/Attacks/Slasher/` (e.g., `SlasherSlash1.asset`). If they don't exist, it logs warnings but the prefab still functions via HitboxManager's timer fallback. No AttackData SOs are created by this task — they'll be authored when Slasher's animations are ready.

---

## File Plan

### New Files

**`Assets/Editor/Characters/SlasherCharacterCreator.cs`**
- Menu item: `TomatoFighters > Characters > Create Slasher`
- Creates/loads `Slasher_MovementConfig.asset` (DD-1 values)
- Creates/loads `Slasher_ComboDefinition.asset` (DD-2 steps)
- Wires AttackData SOs if they exist (DD-7)
- Populates `CharacterPrefabConfig` with DD-3 hitboxes
- Delegates to `PlayerPrefabCreator.CreatePlayerPrefab(config)`
- Saves to `Assets/Prefabs/Player/Slasher.prefab`

**`Assets/Editor/Prefabs/CharacterSelectionWindow.cs`**
- `EditorWindow` subclass, title "Movement Test Scene"
- `CharacterType` enum popup field
- "Create Scene" button → calls `MovementTestSceneCreator.CreateTestScene(type)`
- Window size ~300x100

### Modified Files

**`Assets/Editor/Characters/MysticaCharacterCreator.cs`**
- Change `PREFAB_PATH` from `"Assets/Prefabs/Player/Player.prefab"` to `"Assets/Prefabs/Player/Mystica.prefab"`

**`Assets/Editor/Prefabs/MovementTestSceneCreator.cs`**
- Remove `[MenuItem]` attribute from `CreateTestScene()` (window handles menu now)
- Add `public static void CreateTestScene(CharacterType type)` overload
- Replace hardcoded `PREFAB_PATH` with `$"Assets/Prefabs/Player/{type}.prefab"`
- Remove inline fallback (`BuildInlineFallbackPlayer`) — if prefab missing, log error and return
- Keep `CreateTestScene()` as parameterless calling `CreateTestScene(CharacterType.Mystica)` as default

---

## Execution Order

1. Edit `MysticaCharacterCreator.cs` — change prefab path
2. Create `SlasherCharacterCreator.cs`
3. Edit `MovementTestSceneCreator.cs` — add `CharacterType` parameter, remove fallback
4. Create `CharacterSelectionWindow.cs`

---

## Testing

1. Run `TomatoFighters > Characters > Create Mystica` → verify `Mystica.prefab` created
2. Run `TomatoFighters > Characters > Create Slasher` → verify `Slasher.prefab` created
3. Run `TomatoFighters > Create Movement Test Scene` → window opens
4. Select Mystica → Create Scene → verify Mystica spawns and plays correctly
5. Select Slasher → Create Scene → verify Slasher spawns with correct movement config
6. Select Brutor (no prefab) → Create Scene → verify error log, no crash
