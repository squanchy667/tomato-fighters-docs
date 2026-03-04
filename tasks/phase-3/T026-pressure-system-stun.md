# T026: PressureSystem + Stun

**Phase:** 3 — Defensive Depth
**Type:** Implementation | **Priority:** P0 | **Owner:** Dev 1
**Status:** DONE | **Completed:** 2026-03-04
**Depends on:** T016 (DefenseSystem)
**Branch:** `combat/T026-pressure-system-stun`

---

## Summary

Wire the per-character StunRate (PRS) stat into the damage pipeline so enemy pressure meters fill at character-appropriate rates. Punish hits fill 2x faster. When the meter reaches threshold, the enemy enters a stunned state (configurable ~3s) allowing free juggle. After stun, the enemy gets invulnerability i-frames with white blink. Fire `OnStun`/`OnStunRecovered` events for Roguelite integration and future camera zoom.

## Context

Enemy stun is ~80% implemented in `EnemyBase`. The existing system has:
- `_pressureFill` accumulator + `pressureThreshold` from `EnemyData`
- `AddStun(float amount)` that fills the meter
- `StunRoutine()` coroutine → `stunDuration` timer → `InvulnerabilityBlink()`
- `isPunishDamage ? 2f : 1f` multiplier in `TakeDamage()`

What's missing:
- StunRate stat not wired into the damage pipeline
- `DamagePacket` doesn't carry a `stunFillAmount`
- No `OnStun`/`OnStunRecovered` events in `ICombatEvents`
- `PlayerDamageable.AddStun()` is a no-op stub

## Design Decisions

### DD-1: StunFill Calculated Attacker-Side
**Decision:** HitboxManager calculates `stunFillAmount` and bakes it into `DamagePacket`. EnemyBase just calls `AddStun(packet.stunFillAmount)`.

**Rationale:** The attacker owns their stats. The defender shouldn't need to know about attacker stats — it just receives "how much pressure this hit applies." This also lets enemies have different pressure resistances later without knowing who hit them.

```csharp
// In HitboxManager.BuildDamagePacket():
float stunFill = baseAttack * attackData.damageMultiplier * stunRate;
// isPunish multiplier also applied here
stunFill *= isPunish ? 2f : 1f;
```

### DD-2: No PressureResistance Field
**Decision:** Use higher `pressureThreshold` values for tanky enemies instead of adding a `pressureResistance` multiplier to `EnemyData`.

**Rationale:** `pressureThreshold` already solves "hard to stun" (50 for weaklings, 200 for bosses). Adding a resistance multiplier is a one-line change later if needed. Keep it simple — one knob, not two.

### DD-3: Player Stun Stays as Stub
**Decision:** `PlayerDamageable.AddStun()` remains a no-op. Player stun is deferred to a future task.

**Rationale:** Player stun requires input locking (`CharacterMotor`), combo cancellation (`ComboController`), defense reset (`DefenseSystem`), UI feedback, and recovery animation. That's a separate task. T026 focuses on the offensive pressure loop (player → enemy stun).

### DD-4: Camera Zoom — Event Only
**Decision:** Fire `OnStun` event with position data. No camera zoom implementation.

**Rationale:** The task spec says "(fire event)" — we emit, not consume. Camera belongs to World pillar. We provide the hook; camera subscribes later.

### DD-5: StunRate as Placeholder Field
**Decision:** Add `[SerializeField] private float stunRate = 1.0f` to HitboxManager, matching the existing `baseAttack` placeholder pattern.

**Rationale:** `baseAttack` is already a placeholder "until stat system is wired" (HitboxManager line 33). Adding `stunRate` as a matching placeholder keeps things consistent. Wiring the full stat pipeline (`CharacterStatCalculator`) is an integration pass that should touch both `baseAttack` AND `stunRate` together.

---

## File Plan

### 1. `Scripts/Shared/Data/DamagePacket.cs` (MODIFY)

Add `stunFillAmount` readonly field to the immutable struct.

```csharp
/// <summary>Pre-calculated pressure fill for the target's stun meter.</summary>
public readonly float stunFillAmount;

public DamagePacket(
    DamageType type,
    float amount,
    bool isPunishDamage,
    Vector2 knockbackForce,
    Vector2 launchForce,
    CharacterType source,
    float stunFillAmount)
{
    // ... existing fields ...
    this.stunFillAmount = stunFillAmount;
}
```

**Breaking change:** All existing `new DamagePacket(...)` call sites must add the new parameter. Search for all usages and update.

### 2. `Scripts/Shared/Data/CombatEventData.cs` (MODIFY)

Add two new event data structs:

```csharp
/// <summary>Data for when an enemy becomes stunned (pressure meter full).</summary>
public readonly struct StunEventData
{
    public readonly CharacterType lastHitBy;
    public readonly Vector2 stunnedPosition;
    public readonly float stunDuration;

    public StunEventData(CharacterType lastHitBy, Vector2 stunnedPosition, float stunDuration) { ... }
}

/// <summary>Data for when an enemy recovers from stun.</summary>
public readonly struct StunRecoveredEventData
{
    public readonly Vector2 recoveredPosition;

    public StunRecoveredEventData(Vector2 recoveredPosition) { ... }
}
```

### 3. `Scripts/Shared/Interfaces/ICombatEvents.cs` (MODIFY)

Add two events:

```csharp
/// <summary>Fired when an enemy's pressure meter fills and they become stunned.</summary>
event Action<StunEventData> OnStun;

/// <summary>Fired when an enemy recovers from stun (before invulnerability blink).</summary>
event Action<StunRecoveredEventData> OnStunRecovered;
```

### 4. `Scripts/Combat/Hitbox/HitboxManager.cs` (MODIFY)

- Add `[SerializeField] private float stunRate = 1.0f` placeholder field with tooltip matching `baseAttack` pattern
- Update `BuildDamagePacket()` to calculate `stunFillAmount`:

```csharp
[Header("Damage")]
[Tooltip("Base attack value for damage calculation until stat system is wired.")]
[SerializeField] private float baseAttack = 10f;

[Tooltip("Pressure fill rate multiplier until stat system is wired. " +
         "Maps to PRS stat: Slasher=1.5, Brutor=1.0, Viper=0.8, Mystica=0.5.")]
[SerializeField] private float stunRate = 1.0f;

private DamagePacket BuildDamagePacket(AttackData attackData, bool isPunish = false)
{
    float damage = baseAttack * attackData.damageMultiplier;
    float stunFill = damage * stunRate * (isPunish ? 2f : 1f);

    return new DamagePacket(
        type: DamageType.Physical,
        amount: damage,
        isPunishDamage: isPunish,
        knockbackForce: attackData.knockbackForce,
        launchForce: attackData.launchForce,
        source: comboController != null ? comboController.CharacterType : CharacterType.Brutor,
        stunFillAmount: stunFill
    );
}
```

**Note:** `isPunish` parameter added to `BuildDamagePacket()`. Currently always `false` — punish detection is wired in a future task when enemy punish windows are fully integrated. The 2x multiplier logic is ready for when `isPunish = true` is passed.

### 5. `Scripts/World/EnemyBase.cs` (MODIFY)

Update `TakeDamage()` to use `packet.stunFillAmount` instead of recalculating:

```csharp
public void TakeDamage(DamagePacket damage)
{
    if (_isDead || _isInvulnerable) return;

    _currentHealth -= damage.amount;

    // Use pre-calculated stun fill from attacker's stats (DD-1)
    AddStun(damage.stunFillAmount);

    ApplyKnockback(damage.knockbackForce);
    ApplyLaunch(damage.launchForce);
    OnDamaged(damage);

    if (_currentHealth <= 0f)
    {
        _currentHealth = 0f;
        Die();
    }
}
```

Track `_lastHitBy` for stun event data:

```csharp
private CharacterType _lastHitBy;

public void TakeDamage(DamagePacket damage)
{
    // ... existing logic ...
    _lastHitBy = damage.source;
    // ...
}
```

Fire events from stun lifecycle. Add an `OnStunTriggered` event for external subscribers:

```csharp
/// <summary>Fired when this enemy becomes stunned. Camera/UI/Roguelite subscribe.</summary>
public event Action<StunEventData> StunTriggered;

/// <summary>Fired when this enemy recovers from stun.</summary>
public event Action<StunRecoveredEventData> StunRecovered;

private IEnumerator StunRoutine()
{
    _isStunned = true;
    OnStunned();

    StunTriggered?.Invoke(new StunEventData(
        _lastHitBy, transform.position, enemyData.stunDuration));

    yield return new WaitForSeconds(enemyData.stunDuration);

    _isStunned = false;
    _pressureFill = 0f;
    OnRecovery();

    StunRecovered?.Invoke(new StunRecoveredEventData(transform.position));

    yield return InvulnerabilityBlink();
}
```

### 6. `Scripts/Combat/PlayerDamageable.cs` (MODIFY)

Add TODO comment on the stub:

```csharp
/// <inheritdoc/>
public void AddStun(float amount)
{
    // TODO: Player stun — requires input lock, combo cancel, defense reset, UI. Separate task.
}
```

### 7. Update all `DamagePacket` call sites (MODIFY)

Search codebase for `new DamagePacket(` and add `stunFillAmount: 0f` to any call sites outside HitboxManager (e.g., enemy attacks, test code). Enemy attacks against the player should pass `stunFillAmount: 0f` since player stun is stubbed.

---

## Acceptance Criteria

- [x] `DamagePacket` carries `stunFillAmount` field
- [ ] HitboxManager calculates `stunFillAmount = damage * stunRate * punishMultiplier`
- [ ] `stunRate` placeholder field on HitboxManager (default 1.0)
- [ ] EnemyBase uses `packet.stunFillAmount` for pressure fill (not recalculating)
- [ ] `OnStun` and `OnStunRecovered` events added to `ICombatEvents`
- [ ] EnemyBase fires `StunTriggered`/`StunRecovered` events
- [ ] `StunEventData` includes `lastHitBy`, position, duration
- [ ] Player `AddStun()` remains stub with TODO comment
- [ ] All existing `DamagePacket` call sites updated
- [ ] Stun → invulnerability blink flow preserved (already works)
- [ ] Per-character StunRate values documented in field tooltip

## Execution Order

1. `DamagePacket.cs` — add field (breaks call sites)
2. Fix all `DamagePacket` call sites — add `stunFillAmount: 0f`
3. `CombatEventData.cs` — add event structs
4. `ICombatEvents.cs` — add event signatures
5. `HitboxManager.cs` — add `stunRate` field + update `BuildDamagePacket()`
6. `EnemyBase.cs` — use `stunFillAmount`, track `_lastHitBy`, fire events
7. `PlayerDamageable.cs` — update TODO comment

## Testing Notes

- Set `stunRate = 1.5` in Inspector on HitboxManager to simulate Slasher
- Enemy with `pressureThreshold = 30` should stun in ~3 hits at 10 base damage
- Verify stun → invulnerability blink → pressure reset cycle
- Verify `StunTriggered` event fires (add temporary `Debug.Log` subscriber)
- Verify `isPunishDamage = true` packet fills pressure 2x faster
