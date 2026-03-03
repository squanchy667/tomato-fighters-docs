# T014: ComboSystem — All 4 Characters

**Phase:** 2 | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T004, T005
**Status:** DONE | **Completed:** 2026-03-03 | **Branch:** tal

---

## Summary

Extend the combo system to all 4 characters by creating ComboDefinition, AttackData, and ComboInteractionConfig ScriptableObject assets. No code changes to the combo infrastructure — this is a data authoring task using editor scripts.

## Acceptance Criteria

- [x] 4 distinct combo trees (ComboDefinition SOs)
- [x] Character-specific timing windows
- [x] All AttackData assets created (~25 total)
- [x] Skill/Heavy attacks per character
- [x] ComboInteractionConfig per character with tuned cancel behavior
- [x] Editor script to create all assets programmatically

## Design Decisions

### DD-1: Unified editor script
One `CreateAllCharacterAttacks.cs` with a single menu item that creates all AttackData assets for Brutor, Slasher, and Viper. Existing `CreateMysticaAttacks.cs` stays as-is (it skips duplicates).

### DD-2: Combo tree structures

**Brutor (7 steps — exists from T004):**
```
Light → L1 → L2 → L3 (sweep finisher)
              └→ H  (launcher)
Heavy → H1 → H2 (ground pound finisher)
```

**Slasher (8 steps) — deepest tree, most cancel options:**
```
Light → L1 → L2 → L3 → L4 (spinning finisher)
              └→ H  (lunge thrust, wall-bounce)
Heavy → H1 → H2 (lunge finisher)
         └→ L  (quick slash into light chain)
```
- Tightest combo windows (~0.25s)
- All steps allow dash-cancel on hit
- Heavy branch can re-enter light chain (unique to Slasher)

**Mystica (6 steps) — shallow tree, forgiving windows:**
```
Light → L1 → L2 → L3 (burst finisher)
Heavy → H1 (Arcane Bolt) → H2 (empowered bolt finisher)
```
- Widest combo windows (~0.4s)
- No cross-branching
- Jump-cancel priority (blink escape)

**Viper (6 steps) — medium depth, ranged flavor:**
```
Light → L1 → L2 → L3 (rapid burst finisher)
              └→ H  (charged shot)
Heavy → H1 (charged shot) → H2 (piercing finisher)
```
- Medium combo windows (~0.3s)
- Light chain branches into heavy
- Dash-cancel on hit (backflip retreat)

### DD-3: AttackData tuning values

| Character | Light Multipliers | Heavy Multipliers | Finisher Mult | Knockback | Total Frames |
|-----------|------------------|-------------------|---------------|-----------|--------------|
| **Brutor** | 0.8 → 0.9 → 1.0 | 1.2 → 1.5 | 1.3 (sweep), 1.5 (ground pound) | High (3-5) | Slow (20-30f) |
| **Slasher** | 0.6 → 0.7 → 0.8 → 0.9 | 1.0 → 1.3 | 1.2 (spin), 1.4 (lunge) | Low (1-2) | Fast (12-18f) |
| **Mystica** | 0.6 → 0.8 → 1.0 | 1.4 → 1.6 | 1.2 (burst), 1.6 (empowered bolt) | Low-Med (1.5-2.5) | Medium (16-22f) |
| **Viper** | 0.7 → 0.8 → 0.9 | 1.2 → 1.5 | 1.1 (rapid), 1.5 (piercing) | Low (1-1.5) | Medium-Fast (14-20f) |

Special properties:
- Brutor Slam (H1): `isOTGCapable = true`
- Brutor Sweep finisher: `causesWallBounce = true`
- Slasher Lunge (H1): `causesWallBounce = true`
- Slasher L4 spinning finisher: `causesLaunch = true`

Frame values are design intent — real tuning happens when animation clips land (T024).

### DD-4: ComboInteractionConfig per character

| Config | Brutor | Slasher | Mystica | Viper |
|--------|--------|---------|---------|-------|
| cancelPriority | DashOverJump | DashOverJump | JumpOverDash | DashOverJump |
| dashCancelResetsCombo | true | **false** | true | true |
| jumpCancelResetsCombo | false | false | false | false |
| resetOnStagger | true | true | true | true |
| resetOnDeath | true | true | true | true |
| lockMovementDuringAttack | true | true | true | **false** |
| lockMovementDuringFinisher | true | true | true | true |

- **Slasher**: dash-cancel doesn't reset combo (can dash mid-chain and continue)
- **Mystica**: jump-cancel priority (blink escape)
- **Viper**: can move during normal attacks (kiting while shooting)

## File Plan

| File | Action | Description |
|------|--------|-------------|
| `Editor/CreateAllCharacterAttacks.cs` | CREATE | Unified editor script for Brutor/Slasher/Viper AttackData assets |
| `Editor/CreateComboDefinitions.cs` | CREATE | Editor script for ComboDefinition + ComboInteractionConfig assets |
| `ScriptableObjects/Attacks/Brutor/*.asset` | GENERATED | 7 AttackData assets (via editor menu) |
| `ScriptableObjects/Attacks/Slasher/*.asset` | GENERATED | 8 AttackData assets (via editor menu) |
| `ScriptableObjects/Attacks/Viper/*.asset` | GENERATED | 6 AttackData assets (via editor menu) |
| `ScriptableObjects/ComboDefinitions/*.asset` | GENERATED | 3 new ComboDefinition assets (via editor menu) |
| `ScriptableObjects/ComboInteractionConfigs/*.asset` | GENERATED | 4 ComboInteractionConfig assets (via editor menu) |

## Execution Order

1. `CreateAllCharacterAttacks.cs` — all AttackData assets for Brutor, Slasher, Viper
2. `CreateComboDefinitions.cs` — all ComboDefinition + ComboInteractionConfig assets
3. Verify compilation with zero warnings

## Dependencies

- **Depends on:** T004 (ComboStateMachine infrastructure), T005 (AttackData SO + Mystica attacks)
- **Blocks:** T015 (HitboxManager), T016 (DefenseSystem), T017 (Character Passives)
