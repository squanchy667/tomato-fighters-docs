# T036: OTG vs TechHit System

> **Type:** implementation | **Priority:** P2 | **Owner:** Dev 1 | **Depends on:** T027
> **Status:** DONE | **Completed:** 2026-03-06 | **Branch:** tal

## Summary

Add hit-gating logic to HitboxManager that enforces OTG vs TechHit rules on downed enemies. When an enemy is in the OTG state (knocked down), only attacks with `isOTGCapable = true` (Arcana/path abilities) deal damage. Normal attacks are blocked and show an "immune" visual indicator. During TechRecover, all attacks are blocked.

## Design Decisions

### DD-1: OTG timer does NOT refresh on hit
- OTG-capable hits during the OTG window deal damage but do not reset the 1.0s OTG timer
- Creates natural combo endpoints — player has a fixed window to land Arcana hits
- If they want more, they relaunch the enemy (via `JuggleSystem.Relaunch()`)
- Rationale: Prevents infinite OTG loops; encourages varied combo routing

### DD-2: Blocked OTG hits show a new "immune" indicator
- When a non-OTG-capable attack hits a downed enemy, a distinct visual cue plays (brief icon/flash)
- NOT silent — the player gets clear feedback that "this attack can't hit right now"
- Placeholder implementation in T036 (sprite-based); can be upgraded to full particle VFX in T053

### DD-3: OTG and TechRecover have distinct visuals
- **OTG state:** Enemy sprite gets a color tint (e.g., darker/reddish) signaling "window is open for Arcana"
- **TechRecover state:** Enemy plays invuln blink (white flash loop), reusing the pattern from EnemyBase stun recovery
- Player can visually distinguish "hit me with Arcana" from "don't bother"

### DD-4: Blocked hit feedback via `NotifyBlockedHit()` on IJuggleTarget
- HitboxManager calls `IJuggleTarget.NotifyBlockedHit()` when a hit is gated
- The entity (EnemyBase/JuggleSystem) handles its own visual feedback
- Follows the same pattern as `ApplyKnockback()` and `ApplyLaunch()` — Combat tells the target what happened, target decides how to show it
- No cross-pillar VFX references, no new event bus needed

### DD-5: Scope boundary
**In scope:**
- OTG/TechRecover hit gating in HitboxManager (~15-20 lines)
- `NotifyBlockedHit()` on IJuggleTarget interface + implementation in JuggleSystem
- "Immune" visual feedback (simple sprite-based flash/icon)
- OTG color tint on downed enemies
- TechRecover invuln blink (reuse existing EnemyBase stun recovery pattern)

**Deferred:**
- Full particle VFX for immune indicator -> T053 (Game Feel Pass)
- Balancing OTG/TechRecover timing -> T054 (Combat Balancing)
- Marking path abilities as `isOTGCapable` -> already done in T028/T037
- Relaunch-on-OTG-hit -> already exists via `JuggleSystem.Relaunch()`

## File Plan

### Modified Files

#### 1. `Scripts/Combat/Hitbox/HitboxManager.cs`
- In `HandleHitDetected()`, add OTG/TechRecover check before damage application
- Query `target as IJuggleTarget`, check `CurrentJuggleState`
- If OTG + `!attackData.isOTGCapable` -> call `NotifyBlockedHit()`, return (no damage)
- If TechRecover -> call `NotifyBlockedHit()`, return (no damage)
- Otherwise -> normal hit resolution

```csharp
// Before existing damage application logic:
if (target is IJuggleTarget juggleTarget)
{
    var state = juggleTarget.CurrentJuggleState;
    if (state == JuggleState.OTG && !attackData.isOTGCapable)
    {
        juggleTarget.NotifyBlockedHit();
        return;
    }
    if (state == JuggleState.TechRecover)
    {
        juggleTarget.NotifyBlockedHit();
        return;
    }
}
```

#### 2. `Scripts/Shared/Interfaces/IJuggleTarget.cs`
- Add `void NotifyBlockedHit()` method to interface

#### 3. `Scripts/Combat/Juggle/JuggleSystem.cs`
- Implement `NotifyBlockedHit()` — triggers "immune" visual
- Add OTG color tint logic: on entering OTG state, apply tint to SpriteRenderer; remove on exit
- Add TechRecover blink logic: reuse invuln blink pattern from EnemyBase

#### 4. `Scripts/World/EnemyBase.cs`
- Subscribe to JuggleSystem's OTG/TechRecover state changes for visual feedback
- Wire up `NotifyBlockedHit()` to spawn the "immune" indicator
- Apply OTG tint and TechRecover blink via SpriteRenderer

## Execution Order

1. `IJuggleTarget.cs` — add `NotifyBlockedHit()` to interface
2. `JuggleSystem.cs` — implement `NotifyBlockedHit()`, add OTG tint + TechRecover blink
3. `HitboxManager.cs` — add hit-gating logic in `HandleHitDetected()`
4. `EnemyBase.cs` — wire up visual feedback for blocked hits and state tints

## Acceptance Criteria

- [ ] Normal attacks on OTG enemies are blocked (no damage applied)
- [ ] OTG-capable attacks on OTG enemies deal damage normally
- [ ] All attacks on TechRecover enemies are blocked
- [ ] Blocked hits show "immune" visual indicator on the target
- [ ] OTG enemies have a distinct color tint
- [ ] TechRecover enemies show invuln blink
- [ ] OTG timer does NOT refresh when hit by OTG-capable attack
- [ ] State flow: OTG (1.0s) -> TechRecover (0.4s) -> Grounded works correctly
- [ ] Compiles with zero warnings
