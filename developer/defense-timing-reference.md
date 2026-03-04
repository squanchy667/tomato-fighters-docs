# Defense Timing Reference

Developer reference for how deflect, clash, and dodge timing windows interact with attack hitboxes. Read this before tuning `AttackData` or `DefenseConfig` values.

---

## Core Concept: Two Independent Timelines

Defense windows and attack hitboxes belong to **different characters** and are ticked by **different systems**. A defense outcome is determined by a **snapshot** of the defender's state at the exact moment `OnTriggerEnter2D` fires.

```
ATTACKER:  [--startup--][===HITBOX ACTIVE===][--recovery--]
                         collider ON, can trigger hits

DEFENDER:  [input]...[==DEFENSE WINDOW==]...
                     state checked at moment of collision
```

There is no buffering or retroactive resolution. Whatever `DefenseState` the defender is in when the hit arrives determines the outcome.

---

## Clash

**Trigger:** Defender presses heavy attack.
**Window:** Opens after `clashWindowStart` delay, lasts until `clashWindowEnd` (both measured from heavy input).
**Requirement:** Defender must face the attacker (within 90 degrees).

```
ATTACKER:  [----startup----][===HITBOX===]

DEFENDER:  [heavy input][.delay.][==CLASH WINDOW==][...own hitbox later...]
                                 DefenseState = HeavyStartup
```

The attacker's hitbox must arrive **during** the clash window. If it arrives during the delay or after the window closes, it's a `Hit`.

### Critical Design Constraint

**The defender's clash window must open BEFORE the defender's own hitbox activates.**

If the defender's hitbox fires first, it hits the attacker (potentially causing hitstun), which interrupts the attacker's attack before their hitbox reaches the defender. Result: no clash is possible.

```
CORRECT:   [heavy][.delay.][==CLASH==].......[own hitbox activates]
                           ^ incoming hits resolve as Clash here

BROKEN:    [heavy][own hitbox activates][.delay.][==CLASH==]
                  ^ defender hits attacker first, interrupts them
                    attacker's hitbox never reaches defender
```

**Rule:** For every heavy attack, verify that `clashWindowStart` (seconds) < `hitboxStartFrame / 60` (seconds). Ideally `clashWindowEnd` < hitbox start time too, so the clash window is entirely within startup.

---

## Deflect

**Trigger:** Defender dashes horizontally toward the attacker.
**Window:** Opens **instantly** on dash start, lasts `deflectWindowDuration`.
**Requirement:** Dash direction must point toward attacker (within 90 degrees). Dashing away = `Hit`.

```
ATTACKER:  [--startup--][=======HITBOX=======]

DEFENDER:  .......[dash toward attacker]
                  [==DEFLECT WINDOW==]
                  ^ opens instantly, no delay
```

- No damage on success.
- Does **not** work against `Unstoppable` attacks.

---

## Dodge

**Trigger:** Defender dashes vertically (up or down).
**Window:** Opens after `dodgeIFrameStart` delay, lasts until `dodgeIFrameEnd`.
**Requirement:** Dash direction must be vertical (within 45 degrees of up/down). Direction relative to attacker doesn't matter — pure i-frames.

```
ATTACKER:  [--startup--][=======HITBOX=======]

DEFENDER:  .......[dash up/down]
                  [.delay.][========DODGE I-FRAMES========]
```

- No damage on success.
- **Works against Unstoppable attacks** (only defense that does).

---

## Comparison

| | Deflect | Clash | Dodge |
|---|---------|-------|-------|
| **Trigger** | Dash toward | Heavy attack | Dash vertical |
| **Delay** | None | `clashWindowStart` | `dodgeIFrameStart` |
| **Duration** | `deflectWindowDuration` | `clashWindowEnd - Start` | `dodgeIFrameEnd - Start` |
| **Direction** | Must face attacker | Must face attacker | Any (pure i-frames) |
| **Beats Unstoppable** | No | No | Yes |
| **Damage on success** | 0% | 50% + 30% knockback | 0% |
| **Bonus** | Character-specific | Character-specific | Character-specific |

---

## Current Timing Values

### Player Clash Windows (generous for testing)

| Character | Clash Start | Clash End | Effective Duration |
|-----------|:-----------:|:---------:|:------------------:|
| Brutor | 0ms | 500ms | 500ms |
| Slasher | 0ms | 400ms | 400ms |
| Mystica | 0ms | 300ms | 300ms |
| Viper | 0ms | 400ms | 400ms |

### Player Deflect Windows

| Character | Duration |
|-----------|:--------:|
| Brutor | 200ms |
| Slasher | 120ms |
| Mystica | 100ms |
| Viper | 140ms |

### Player Dodge Windows

| Character | Delay | End | Effective Duration |
|-----------|:-----:|:---:|:------------------:|
| Brutor | 80ms | 200ms | 120ms |
| Slasher | 40ms | 250ms | 210ms |
| Mystica | 30ms | 350ms | 320ms |
| Viper | 40ms | 300ms | 260ms |

### TestDummy Attack (large for testing)

| Property | Value |
|----------|:-----:|
| Telegraph | 1.0s |
| Hitbox active | 1.5s |
| hitboxStartFrame | 12 (~200ms) |
| hitboxActiveFrames | 18 (~300ms) |
| totalFrames | 45 (~750ms) |

---

## Attack Data Audit Checklist

When creating or tuning a heavy attack's `AttackData`, verify:

1. **Clash before hitbox:** `clashWindowStart` < `hitboxStartFrame / 60fps` (the clash window must be open before the hitbox activates)
2. **No dead zone too large:** gap between `clashWindowEnd` and hitbox start should be reasonable (50ms+ gaps mean the defender is vulnerable with no hitbox active)
3. **Telegraph matches startup:** enemy telegraph duration should roughly match the time before hitbox activation, so the player can read and react

### Formula

```
hitboxStartTime = hitboxStartFrame / (60 * animationSpeed)

PASS if: characterClashWindowStart < hitboxStartTime
IDEAL if: characterClashWindowEnd <= hitboxStartTime
```

---

## Character Defense Bonuses

| Character | Bonus Effect |
|-----------|-------------|
| Brutor | Prevents slideback (zeroes velocity) |
| Slasher | Guaranteed crit on next attack |
| Mystica | Restores 15 mana |
| Viper | Pending projectile reflect |

---

## Source Files

| File | Purpose |
|------|---------|
| `Scripts/Combat/Defense/DefenseSystem.cs` | State machine, window timers, event subscriptions |
| `Scripts/Combat/Defense/DefenseConfig.cs` | Per-entity timing SO |
| `Scripts/Combat/Defense/DefenseResolver.cs` | Pure resolution logic (testable) |
| `Scripts/Combat/Defense/DefenseState.cs` | Enum: None, Dashing, HeavyStartup |
| `Scripts/Combat/Defense/DefenseBonus.cs` | Abstract bonus base |
| `Scripts/Combat/Defense/Bonuses/*.cs` | 4 character-specific bonuses |
| `Scripts/Shared/Enums/DamageResponse.cs` | Hit, Deflected, Clashed, Dodged |
| `Tests/EditMode/Combat/Defense/DefenseResolverTests.cs` | 35+ test scenarios |
