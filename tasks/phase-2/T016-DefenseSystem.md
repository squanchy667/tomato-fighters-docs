# T016: DefenseSystem — Deflect/Clash/Dodge

**Type:** implementation | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T014
**Status:** DONE | **Completed:** 2026-03-04 | **Branch:** tal
**Blocks:** T026 (PressureSystem), T027 (WallBounce/AirJuggle)

---

## Summary

Per-entity defense resolution system. When an attack lands on a target, the target's `IDamageable.ResolveIncoming()` checks current defense state and returns a `DamageResponse` — `Hit`, `Deflected`, `Clashed`, or `Dodged`. Defense actions are **directional**: only effective when facing the attacker.

- **Deflect** — dash toward the attacker, generous window (0–150ms)
- **Clash** — heavy attack facing toward the attacker, tight window (20–80ms)
- **Dodge** — dash vertically (up/down), i-frames (50–300ms)
- **Dash away / attack facing away** — no defensive benefit
- **Unstoppable attacks** bypass deflect and clash (only dodge works)

Both player and enemies can defend. Character-specific bonuses fire on any successful defense.

---

## Design Decisions

### DD-1: Defender-Side Resolution
Each hit is independently resolved against the target's current defense state. No frame-buffering needed — defense actions (dash, heavy attack) are mutually exclusive, so there is no ambiguity about which window is active. When `HitboxManager` detects a collision, it calls `target.ResolveIncoming(attackerPosition)` and acts on the returned `DamageResponse`.

**Rationale:** Clash/deflect/dodge are properties of the defender's state at the moment of impact. Each hit resolves on its own — if player hits an enemy in a clash window, that's a clash; if the enemy hits the player in a deflect window, that's a deflect. No need to correlate hits within the same frame.

### DD-2: Event-Driven Window Activation
`DefenseSystem` subscribes to events from `CharacterMotor` and `ComboController` (e.g., `OnDashStarted`, `OnHeavyAttackStarted`) to open defense windows. Window timers count down in `FixedUpdate` and close automatically.

**Rationale:** Matches existing event patterns in the codebase (`ComboController` already fires `AttackStarted`, `DashCancelTriggered`, etc.). Loose coupling — `DefenseSystem` doesn't poll other systems.

```csharp
void OnEnable() {
    motor.OnDashStarted += HandleDashStarted;
    comboController.OnHeavyAttackStarted += HandleHeavyAttackStarted;
}
```

### DD-3: Directional Defense
All defensive actions require facing the attacker:
- **Deflect** — dash direction points toward attacker position
- **Clash** — attack facing direction points toward attacker position
- **Dodge** — dash direction is vertical (up/down), regardless of attacker position
- **Dash away** — no defensive benefit

**Rationale:** Adds skill expression. The defender must read the attack and choose the correct directional response. Dash toward = deflect (high reward). Dash vertical = dodge (safe). Dash away = escape but no defense benefit.

```csharp
if (isDashing) {
    Vector2 toAttacker = (attackerPos - transform.position).normalized;
    if (IsFacing(dashDirection, toAttacker)) return DamageResponse.Deflected;
    if (IsVertical(dashDirection))           return DamageResponse.Dodged;
    return DamageResponse.Hit; // dashing away
}
if (isInClashWindow) {
    Vector2 toAttacker = (attackerPos - transform.position).normalized;
    if (IsFacing(facingDirection, toAttacker)) return DamageResponse.Clashed;
    return DamageResponse.Hit; // attacking away from threat
}
return DamageResponse.Hit;
```

### DD-4: Strategy Pattern for Defense Bonuses
Character-specific bonuses are implemented as ScriptableObject subclasses of `DefenseBonus`. Each character gets one `DefenseBonus` SO assigned in their `DefenseConfig`. `DefenseSystem` calls `bonus.Apply(context, responseType)` after a successful defense.

**Rationale:** Open/closed — new bonuses are new SO subclasses, `DefenseSystem` never changes. Inspector-assignable — swap a character's bonus without code changes. Easy to test in isolation.

```csharp
public abstract class DefenseBonus : ScriptableObject {
    public abstract void Apply(DefenseContext context, DamageResponse responseType);
}
```

### DD-5: Single Bonus Per Character (For Now)
One `DefenseBonus` SO per character, applied on any successful defense (deflect, clash, or dodge). The `DamageResponse` parameter is passed so bonuses can differentiate later if needed.

**Rationale:** Keeps it simple for initial implementation. The strategy pattern makes it trivial to split into per-action bonuses later (add `DeflectBonus`, `ClashBonus`, `DodgeBonus` fields to `DefenseConfig`).

### DD-6: DefenseSystem on Both Player and Enemies
Both player and enemies get a `DefenseSystem` component. Enemy `DefenseConfig` SOs can have zeroed windows (no defense) or real windows (boss that can clash). Resolution logic is universal.

**Rationale:** The game design supports enemies clashing — when an enemy is hit during its clash window, it deflects. Shared code path means no special-casing.

### DD-7: Resolution Through IDamageable Interface
`IDamageable` gains a `ResolveIncoming(Vector2 attackerPosition)` method that returns `DamageResponse`. `PlayerDamageable` and `EnemyBase` delegate to their `[SerializeField] DefenseSystem` internally. `HitboxManager` calls this before deciding whether to apply damage.

**Rationale:** No `GetComponent` at runtime. Resolution stays behind the interface — `HitboxManager` doesn't need to know about `DefenseSystem` directly.

```csharp
public interface IDamageable {
    DamageResponse ResolveIncoming(Vector2 attackerPosition);
    void TakeDamage(DamagePacket damage);
    // ...existing members...
}
```

---

## Acceptance Criteria

- [ ] Deflect with configurable window per character (0–150ms default)
- [ ] Clash with tighter window (20–80ms default)
- [ ] Dodge i-frames (50–300ms default)
- [ ] All defense actions are directional (toward attacker / vertical)
- [ ] Unstoppable attacks bypass deflect/clash, only dodge works
- [ ] Character-specific defense bonuses (Brutor no-slideback, Slasher crit, Mystica mana, Viper projectile-reflect)
- [ ] DamageResponse enum (Hit, Deflected, Clashed, Dodged)
- [ ] Enemies can also defend (DefenseSystem on both player and enemies)
- [ ] ICombatEvents fired on defense (OnDeflect, OnClash, OnDodge)
- [ ] DefenseResolver has full NUnit test coverage

---

## File Plan

### New Files

#### 1. `Scripts/Shared/Enums/DamageResponse.cs`
```csharp
namespace TomatoFighters.Shared.Enums
{
    public enum DamageResponse { Hit, Deflected, Clashed, Dodged }
}
```

#### 2. `Scripts/Combat/Defense/DefenseConfig.cs`
ScriptableObject with per-entity defense timing configuration.

```csharp
[CreateAssetMenu(menuName = "TomatoFighters/Combat/DefenseConfig")]
public class DefenseConfig : ScriptableObject
{
    [Header("Deflect (Dash Toward)")]
    [Tooltip("Duration in seconds from dash start where deflect is active")]
    [Range(0f, 0.3f)] public float deflectWindowDuration = 0.15f;

    [Header("Clash (Heavy Attack Facing Toward)")]
    [Tooltip("Start delay in seconds from heavy attack start")]
    [Range(0f, 0.1f)] public float clashWindowStart = 0.02f;
    [Tooltip("End time in seconds from heavy attack start")]
    [Range(0f, 0.2f)] public float clashWindowEnd = 0.08f;

    [Header("Dodge (Dash Vertical)")]
    [Tooltip("Start delay in seconds from dash start")]
    [Range(0f, 0.1f)] public float dodgeIFrameStart = 0.05f;
    [Tooltip("End time in seconds from dash start")]
    [Range(0f, 0.5f)] public float dodgeIFrameEnd = 0.3f;

    [Header("Bonus")]
    [Tooltip("Character-specific defense bonus (null = none)")]
    public DefenseBonus defenseBonus;
}
```

#### 3. `Scripts/Combat/Defense/DefenseBonus.cs`
Abstract SO base class for character-specific bonuses.

```csharp
public abstract class DefenseBonus : ScriptableObject
{
    /// <summary>
    /// Called after a successful defense action. Override to apply character-specific effects.
    /// </summary>
    public abstract void Apply(DefenseContext context, DamageResponse responseType);
}

public struct DefenseContext
{
    public GameObject defender;
    public GameObject attacker;
    public Vector2 hitPoint;
    public DamagePacket incomingPacket;
}
```

#### 4. `Scripts/Combat/Defense/Bonuses/BrutorDefenseBonus.cs`
Prevents knockback/slideback on successful defense.

#### 5. `Scripts/Combat/Defense/Bonuses/SlasherDefenseBonus.cs`
Grants critical hit on next attack after defense.

#### 6. `Scripts/Combat/Defense/Bonuses/MysticaDefenseBonus.cs`
Restores mana on successful defense. Amount configurable via serialized field.

#### 7. `Scripts/Combat/Defense/Bonuses/ViperDefenseBonus.cs`
Reflects projectiles on successful defense.

#### 8. `Scripts/Combat/Defense/DefenseResolver.cs`
Plain C# class — pure testable resolution logic. No MonoBehaviour, no Unity dependencies.

```csharp
public class DefenseResolver
{
    /// <summary>
    /// Resolves what happens when an attack reaches a defender.
    /// </summary>
    public DamageResponse Resolve(
        DefenseState currentState,
        Vector2 actionDirection,
        Vector2 toAttackerDirection,
        bool isAttackUnstoppable)
    {
        if (currentState == DefenseState.None)
            return DamageResponse.Hit;

        if (currentState == DefenseState.Dashing)
        {
            if (IsVertical(actionDirection))
                return DamageResponse.Dodged; // dodge: no unstoppable check
            if (isAttackUnstoppable)
                return DamageResponse.Hit;
            if (IsFacing(actionDirection, toAttackerDirection))
                return DamageResponse.Deflected;
            return DamageResponse.Hit; // dashing away
        }

        if (currentState == DefenseState.HeavyStartup)
        {
            if (isAttackUnstoppable)
                return DamageResponse.Hit;
            if (IsFacing(actionDirection, toAttackerDirection))
                return DamageResponse.Clashed;
            return DamageResponse.Hit; // attacking away
        }

        return DamageResponse.Hit;
    }
}
```

#### 9. `Scripts/Combat/Defense/DefenseSystem.cs`
MonoBehaviour that tracks defense state via event subscriptions. Manages window timers in `FixedUpdate`. Delegates resolution to `DefenseResolver`. Applies `DefenseBonus` on success. Fires `ICombatEvents`.

Key responsibilities:
- Subscribe to `OnDashStarted`, `OnHeavyAttackStarted` events
- Track `DefenseState` and window timers
- Expose `Resolve(Vector2 attackerPosition)` called by the entity's `IDamageable` implementation
- Apply `DefenseConfig.defenseBonus` on successful defense
- Fire `OnDeflect` / `OnClash` / `OnDodge` via `ICombatEvents`

#### 10. `Tests/EditMode/Combat/DefenseResolverTests.cs`
NUnit tests for `DefenseResolver`. Test cases:

- Dash toward + normal attack → Deflected
- Dash toward + unstoppable attack → Hit
- Dash vertical + normal attack → Dodged
- Dash vertical + unstoppable attack → Dodged (dodge beats unstoppable)
- Dash away + any attack → Hit
- Heavy startup facing toward + normal attack → Clashed
- Heavy startup facing toward + unstoppable attack → Hit
- Heavy startup facing away → Hit
- No defense state → Hit

### Modified Files

#### 11. `Scripts/Shared/Interfaces/IDamageable.cs`
Add `ResolveIncoming` method:
```csharp
DamageResponse ResolveIncoming(Vector2 attackerPosition);
```

#### 12. `Scripts/Combat/PlayerDamageable.cs`
Implement `ResolveIncoming` — delegate to `[SerializeField] DefenseSystem`.

```csharp
[SerializeField] private DefenseSystem defenseSystem;

public DamageResponse ResolveIncoming(Vector2 attackerPosition)
{
    if (defenseSystem == null) return DamageResponse.Hit;
    return defenseSystem.Resolve(attackerPosition);
}
```

#### 13. `Scripts/World/EnemyBase.cs`
Implement `ResolveIncoming` — delegate to optional `[SerializeField] DefenseSystem`.

```csharp
[SerializeField] private DefenseSystem defenseSystem;

public DamageResponse ResolveIncoming(Vector2 attackerPosition)
{
    if (defenseSystem == null) return DamageResponse.Hit;
    return defenseSystem.Resolve(attackerPosition);
}
```

#### 14. `Scripts/Combat/Hitbox/HitboxManager.cs`
Replace the temporary damage shim. Before applying damage, call `ResolveIncoming`:

```csharp
private void HandleHitDetected(IDamageable target, Vector2 hitPoint)
{
    var attackData = GetCurrentAttackData();
    var response = target.ResolveIncoming(transform.position);

    switch (response)
    {
        case DamageResponse.Hit:
            var packet = BuildDamagePacket(attackData);
            target.TakeDamage(packet);
            comboController?.OnHitConfirmed();
            break;
        case DamageResponse.Deflected:
            // No damage, fire OnDeflect event
            break;
        case DamageResponse.Clashed:
            // Reduced damage, both stagger, fire OnClash event
            break;
        case DamageResponse.Dodged:
            // No damage, attack passes through
            break;
    }

    OnHitProcessed?.Invoke(BuildHitDetectionData(target, attackData, hitPoint, response));
}
```

---

## Execution Order

1. `DamageResponse` enum (no dependencies)
2. `DefenseConfig` SO + `DefenseBonus` abstract SO (no dependencies)
3. `DefenseResolver` plain C# (depends on enum only)
4. `DefenseResolverTests` (validate resolver logic)
5. `DefenseSystem` MonoBehaviour (depends on config, resolver, events)
6. 4x character `DefenseBonus` subclasses (depend on base SO)
7. `IDamageable` modification (add `ResolveIncoming`)
8. `PlayerDamageable` + `EnemyBase` modifications (implement new method)
9. `HitboxManager` modification (call `ResolveIncoming`, remove temp shim)

---

## Risks and Gotchas

- **Event dependency on CharacterMotor:** `OnDashStarted` event may not exist yet — verify or add it. Same for `OnHeavyAttackStarted` on `ComboController`.
- **Dodge i-frames vs IsInvulnerable:** Dodge sets `IsInvulnerable = true` for the i-frame duration. Make sure this doesn't conflict with other invulnerability sources (stun recovery blink on `EnemyBase`).
- **Projectile reflect (Viper):** The bonus needs a reference to the incoming projectile GameObject to reflect it. This may require a richer `DefenseContext` or deferred implementation if projectiles aren't built yet.
- **Editor script updates:** `PlayerPrefabCreator` and `TestDummyPrefabCreator` need updating to add `DefenseSystem` component and wire the `DefenseConfig` SO reference.
- **ICombatEvents firing:** `DefenseSystem` needs a reference to the entity's `ICombatEvents` implementation to fire `OnDeflect`/`OnClash`/`OnDodge`. Decide whether `DefenseSystem` fires these directly or delegates back to a combat event dispatcher.
