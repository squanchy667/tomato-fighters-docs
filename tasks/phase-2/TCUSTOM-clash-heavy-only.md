# TCUSTOM: Clash Windows — Heavy Attacks Only

**Pillar:** Combat (Dev 1)
**Phase:** 2
**Priority:** Medium
**Status:** DONE
**Completed:** 2026-03-05
**Branch:** tal
**Dependencies:** T016 (DefenseSystem), T014 (ComboSystem)

---

## Summary

Light (LMB) attacks should never open a clash window. Only Heavy attacks and Heavy finishers cause clash. The behavior is data-driven: each AttackData SO declares its own clash window values. Light attacks get `0/0` (no clash), Heavy attacks get character-appropriate windows.

Slasher is the best clasher and keeps the widest windows. Other characters get progressively smaller windows.

---

## Deliverables

1. **`BrutorCharacterCreator.cs`** — expand `WireAttackDataIntoComboSteps` mapping to include per-attack clash window values
2. **`MysticaCharacterCreator.cs`** — same
3. **`ViperCharacterCreator.cs`** — same

No runtime code changes. `DefenseSystem.HandleAttackStarted` already gates on `AttackData.HasClashWindow`, so zeroing out Light attacks' clash windows is sufficient.

---

## Design Decisions

### DD-1: Data-driven approach (no system-level filter)

Instead of adding `if (attackType != AttackType.Heavy) return;` in `DefenseSystem.HandleAttackStarted`, we set clash window data to `0/0` on Light AttackData SOs and real values on Heavy AttackData SOs. The existing `HasClashWindow` check (`clashWindowEnd > clashWindowStart`) is the only gate needed.

**Rationale:** Every attack already carries its own config. Keeping the behavior in the data means a designer could give a specific Light attack a clash window in the future without touching code. The system stays generic.

### DD-2: Clash window hierarchy

Slasher is the best clasher. Other characters get smaller windows, ranked by melee focus:

| Rank | Character | Role | Window Range | Notes |
|------|-----------|------|-------------|-------|
| 1st | Slasher | Melee assassin | 0.25–0.40s | Already set, unchanged |
| 2nd | Brutor | Tank brawler | 0.25–0.30s | Face-to-face fighter, decent clash |
| 3rd | Viper | Ranged | 0.15–0.20s | Less melee focus |
| 4th | Mystica | Mage | 0.15s | Worst physical clasher |

### DD-3: Constraint — clash window must open before hitbox fires

`AttackData.OnValidate` enforces `clashWindowStart < hitboxStartTime`. All proposed values satisfy this trivially since `clashWindowStart = 0` and all hitbox start frames are > 0.

---

## File Plan

### 1. `Assets/Editor/Characters/BrutorCharacterCreator.cs`

**Change:** Expand `WireAttackDataIntoComboSteps` mapping tuple from `(soName, hitboxId)` to `(soName, hitboxId, clashStart, clashEnd)`. Add clash window patching in the loop (same pattern as `SlasherCharacterCreator.cs` lines 353-358).

```csharp
// Step index → (attack SO name, hitboxId, clashStart, clashEnd)
// Light attacks: 0/0 = no clash. Heavy attacks get clash window before hitbox.
var mapping = new (string soName, string hitboxId, float clashStart, float clashEnd)[]
{
    ("BrutorShieldBash1",  "Jab",      0f, 0f),    // 0: light — no clash
    ("BrutorShieldBash2",  "Jab",      0f, 0f),    // 1: light — no clash
    ("BrutorSweep",        "Sweep",    0f, 0f),    // 2: light finisher — no clash
    ("BrutorLauncher",     "Uppercut", 0f, 0.25f), // 3: heavy — hitbox frame 5 (~83ms)
    ("BrutorLauncherSlam", "Slam",     0f, 0.30f), // 4: heavy finisher — hitbox frame 6 (~100ms)
    ("BrutorOverheadSlam", "Slam",     0f, 0.25f), // 5: heavy — hitbox frame 6 (~100ms)
    ("BrutorGroundPound",  "Slam",     0f, 0.30f), // 6: heavy finisher — hitbox frame 7 (~117ms)
};
```

Add clash patching inside the loop:

```csharp
if (attack.clashWindowStart != clashStart || attack.clashWindowEnd != clashEnd)
{
    attack.clashWindowStart = clashStart;
    attack.clashWindowEnd = clashEnd;
    attackDirty = true;
}
```

### 2. `Assets/Editor/Characters/MysticaCharacterCreator.cs`

**Change:** Same pattern as Brutor.

```csharp
var mapping = new (string soName, string hitboxId, float clashStart, float clashEnd)[]
{
    ("MysticaStrike1",       "Burst",    0f, 0f),    // 0: light — no clash
    ("MysticaStrike2",       "Burst",    0f, 0f),    // 1: light — no clash
    ("MysticaStrike3",       "BigBurst", 0f, 0f),    // 2: light finisher — no clash
    ("MysticaArcaneBolt",    "Bolt",     0f, 0.15f), // 3: heavy — hitbox frame 6 (~100ms)
    ("MysticaEmpoweredBolt", "Bolt",     0f, 0.15f), // 4: heavy finisher — hitbox frame 7 (~117ms)
};
```

### 3. `Assets/Editor/Characters/ViperCharacterCreator.cs`

**Change:** Same pattern as Brutor.

```csharp
var mapping = new (string soName, string hitboxId, float clashStart, float clashEnd)[]
{
    ("ViperShot1",        "Lunge", 0f, 0f),    // 0: light — no clash
    ("ViperShot2",        "Lunge", 0f, 0f),    // 1: light — no clash
    ("ViperRapidBurst",   "Sweep", 0f, 0f),    // 2: light finisher — no clash
    ("ViperQuickCharged", "Lunge", 0f, 0.15f), // 3: heavy — hitbox frame 3 (~50ms)
    ("ViperChargedShot",  "Lunge", 0f, 0.20f), // 4: heavy — hitbox frame 5 (~83ms)
    ("ViperPiercingShot", "Lunge", 0f, 0.20f), // 5: heavy finisher — hitbox frame 5 (~83ms)
};
```

### 4. `SlasherCharacterCreator.cs` — NO CHANGES

Already correct. Light attacks have `0f, 0f` and Heavy attacks have `0.25–0.40s` windows.

Reference (current values, unchanged):
```
("SlasherSlash1",        "Slash",     0f,   0f),     // light — no clash
("SlasherSlash2",        "Slash",     0f,   0f),     // light — no clash
("SlasherSlash3",        "WideSlash", 0f,   0f),     // light — no clash
("SlasherHeavySlash",    "WideSlash", 0f,   0.35f),  // heavy
("SlasherLunge",         "Lunge",     0f,   0.35f),  // heavy
("SlasherLungeFinisher", "Lunge",     0f,   0.4f),   // heavy finisher
("SlasherQuickSlash",    "Slash",     0f,   0.25f),  // heavy (re-entry)
("SlasherSpinFinisher",  "Spin",      0f,   0.35f),  // heavy (AoE)
```

---

## Execution Order

1. `BrutorCharacterCreator.cs` — expand mapping + add clash patching
2. `MysticaCharacterCreator.cs` — expand mapping + add clash patching
3. `ViperCharacterCreator.cs` — expand mapping + add clash patching
4. Run all 3 character creator scripts in Unity (`Tools > TomatoFighters > Create Brutor/Mystica/Viper Character`) to regenerate SOs

---

## Verification

After running the creator scripts:
- Light AttackData SOs should have `clashWindowStart=0, clashWindowEnd=0` (`HasClashWindow` = false)
- Heavy AttackData SOs should have `clashWindowEnd > 0` (`HasClashWindow` = true)
- Existing `ClashTimingTests` should still pass (they test `DefenseResolver` and `OpenClashWindow`, not `HandleAttackStarted`)
- In-game: pressing Light (LMB) should NOT give the player clash protection. Only Heavy (RMB) should.
