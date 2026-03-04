# T027: WallBounce + AirJuggle

**Type:** implementation | **Priority:** P1 | **Owner:** Dev 1 | **Depends on:** T015
**Depended on by:** T036 (OTG vs TechHit System)
**Branch:** `pillar1/T027-wall-bounce-air-juggle`

---

## Summary

Two new Combat pillar systems that add depth to the combo game:

- **WallBounceHandler** — Detects when a knocked-back entity hits a wall and bounces them. Each bounce deals minor velocity-based damage (no pressure). Unlimited per combo.
- **JuggleSystem** — Tracks enemy airborne state after launch attacks. Simulates gravity-driven fall (belt-scroll style, like CharacterMotor). Gale element extends airtime via a gravity multiplier in DamagePacket. Sets up the full `JuggleState` enum that T036 will build on.

---

## Design Decisions

### DD-1: Airborne Tracking Architecture
**Decision:** JuggleSystem as a separate MonoBehaviour on enemies.
**Rationale:** Keeps EnemyBase focused on health/damage/stun. JuggleSystem manages its own `simulatedHeight`, `fallVelocity`, and sprite offset — mirrors how CharacterMotor handles player airborne state. Testable independently.

### DD-2: Knockback State Detection
**Decision:** Boolean `IsInKnockback` flag on EnemyBase.
**Rationale:** Set in `ApplyKnockback()`, cleared in `KnockbackRecovery()`. WallBounceHandler checks this flag in `OnCollisionEnter2D`. Simple, explicit, no threshold tuning. The flag also feeds naturally into JuggleSystem state decisions.

### DD-3: Wall Bounce Damage Calculation
**Decision:** Velocity-based: `damage = velocity.magnitude * wallBounceDamageMultiplier`.
**Rationale:** Harder knockback = faster travel = more bounce damage. Self-balancing. The multiplier lives on JuggleConfig SO for tuning. No need to store "original hit damage" state. No pressure added per spec.

### DD-4: Gale Airtime Extension
**Decision:** Add `float juggleGravityMultiplier` field to DamagePacket.
**Rationale:** Keeps JuggleSystem self-contained — it reads the multiplier on launch and applies it to gravity. Default 1.0f; Gale sets it to e.g. 0.6f for slower fall. The actual Gale ritual calculation happens in the Roguelite pillar when building the packet. No cross-pillar reference at runtime.

### DD-5: JuggleState Enum Scope
**Decision:** Define the full enum now: `Grounded`, `Launched`, `Falling`, `Downed`, `Recovering`.
**Rationale:** Defining all states upfront is cheap, documents the intended state flow, and means T036 doesn't need to touch shared enum files. T027 implements `Grounded`/`Launched`/`Falling` transitions. `Downed`/`Recovering` exist as values but aren't transitioned into until T036.

### DD-6: Tuning Constants Location
**Decision:** New `JuggleConfig` ScriptableObject.
**Rationale:** Single SO for both WallBounceHandler and JuggleSystem tuning. Consistent with project pattern (MovementConfig, DefenseConfig). Per-enemy variation already flows through `EnemyData.knockbackResistance` which affects force before JuggleSystem sees it.

### DD-7: Post-Stun Invulnerable Landing
**Decision:** JuggleSystem controls the timing, calls EnemyBase hook on land.
**Rationale:** JuggleSystem already knows exactly when landing happens. When stun ends and enemy is airborne, JuggleSystem intercepts and delays invulnerability blink until `Falling → Grounded` transition. EnemyBase exposes `RequestDelayedInvulnerability()` hook. Clean ownership — landing logic lives in the landing system.

---

## File Plan

### 1. `Scripts/Shared/Enums/JuggleState.cs` (NEW)

```csharp
namespace TomatoFighters.Shared.Enums
{
    /// <summary>
    /// Airborne/grounded state for entities that can be launched and juggled.
    /// Grounded/Launched/Falling implemented by T027.
    /// Downed/Recovering behavior added by T036 (OTG vs TechHit).
    /// </summary>
    public enum JuggleState
    {
        Grounded,    // On the ground, normal state
        Launched,    // Rising from a launch attack
        Falling,     // Past apex, falling back down
        Downed,      // Hit the ground after juggle (T036)
        Recovering   // Getting up from downed (T036)
    }
}
```

### 2. `Scripts/Shared/Data/DamagePacket.cs` (MODIFY)

Add one field to the readonly struct:

```csharp
/// <summary>
/// Gravity multiplier for juggle system. Default 1.0f.
/// Gale element sets this below 1.0 for slower fall (extended airtime).
/// </summary>
public readonly float juggleGravityMultiplier;
```

Update constructor to accept the new parameter (default 1.0f).

### 3. `Scripts/Shared/Data/JuggleConfig.cs` (NEW)

```csharp
[CreateAssetMenu(menuName = "TomatoFighters/JuggleConfig")]
public class JuggleConfig : ScriptableObject
{
    [Header("Juggle Physics")]
    [Tooltip("Gravity applied to launched entities (units/s^2).")]
    [Range(10f, 80f)]
    public float juggleGravity = 40f;

    [Tooltip("Minimum launch velocity to enter Launched state.")]
    [Range(1f, 10f)]
    public float minLaunchVelocity = 2f;

    [Header("Wall Bounce")]
    [Tooltip("Fraction of velocity retained after bouncing off a wall (0-1).")]
    [Range(0.1f, 0.9f)]
    public float bounceVelocityRetention = 0.6f;

    [Tooltip("Minimum velocity to trigger a wall bounce. Below this, entity just stops.")]
    [Range(0.5f, 5f)]
    public float minBounceVelocity = 1.5f;

    [Tooltip("Damage per unit of impact velocity on wall bounce.")]
    [Range(0.1f, 5f)]
    public float wallBounceDamageMultiplier = 0.5f;

    [Header("Landing")]
    [Tooltip("Invulnerability duration after landing from a juggle (seconds).")]
    [Range(0.3f, 2f)]
    public float landingInvulnerabilityDuration = 0.5f;
}
```

### 4. `Scripts/Combat/JuggleSystem.cs` (NEW)

MonoBehaviour added to launchable entities (enemies).

**Responsibilities:**
- Simulated height tracking (`simulatedHeight`, `fallVelocity`) — belt-scroll style, no physics gravity
- Sprite offset (move sprite child up/down based on simulatedHeight)
- State machine: `Grounded → Launched → Falling → Grounded`
- Apply `juggleGravityMultiplier` from DamagePacket to gravity (Gale support)
- On landing: fire `OnLanded` event, trigger invulnerability if post-stun
- Expose `JuggleState CurrentState`, `bool IsAirborne`, `float SimulatedHeight`

**Key methods:**
```csharp
/// <summary>Launch the entity into the air with the given force.</summary>
public void Launch(Vector2 launchForce, float gravityMultiplier = 1f)

/// <summary>Check if post-stun invulnerability should trigger on next landing.</summary>
public void RequestInvulnerabilityOnLanding()

// Called in FixedUpdate:
private void UpdateJuggle()  // simulate gravity, update height
private void UpdateState()   // transition Launched→Falling→Grounded
private void UpdateSpriteOffset()  // move sprite transform
```

**State transitions:**
```
Launch() called with sufficient force
    → Launched (rising)

fallVelocity becomes negative (past apex)
    → Falling

simulatedHeight <= 0 (hit ground)
    → Grounded
    → fire OnLanded
    → if pendingInvulnerability: call EnemyBase.TriggerInvulnerabilityBlink()
```

### 5. `Scripts/Combat/WallBounceHandler.cs` (NEW)

MonoBehaviour added to knockback-able entities.

**Responsibilities:**
- Detect wall collision via `OnCollisionEnter2D` (walls are on a wall layer/tagged)
- Check `EnemyBase.IsInKnockback` — only bounce if actively being knocked back
- Check velocity against `minBounceVelocity` — below threshold, don't bounce
- Reflect velocity: reverse the component normal to the wall, multiply by `bounceVelocityRetention`
- Apply minor damage: `velocity.magnitude * wallBounceDamageMultiplier` — bypass pressure (not a "hit")
- Fire event for VFX/SFX hooks: `OnWallBounce(Vector2 bouncePoint, float impactVelocity)`

**Key methods:**
```csharp
private void OnCollisionEnter2D(Collision2D collision)
{
    if (!IsWall(collision)) return;
    if (!enemyBase.IsInKnockback) return;

    float impactSpeed = rb.velocity.magnitude;
    if (impactSpeed < config.minBounceVelocity) return;

    // Reflect velocity off wall normal
    Vector2 normal = collision.contacts[0].normal;
    Vector2 reflected = Vector2.Reflect(rb.velocity, normal);
    rb.velocity = reflected * config.bounceVelocityRetention;

    // Minor damage, no pressure
    float bounceDamage = impactSpeed * config.wallBounceDamageMultiplier;
    ApplyBounceDamage(bounceDamage);

    OnWallBounce?.Invoke(collision.contacts[0].point, impactSpeed);
}
```

**Wall detection:** Check collision GameObject's layer or tag. The test scene already has wall colliders — use a "Wall" layer (or check for `StaticCollider` tag). Will need to ensure walls have colliders that generate `OnCollisionEnter2D` (not triggers).

### 6. `Scripts/World/EnemyBase.cs` (MODIFY)

Add the following:

```csharp
// ── Knockback State ──────────────────────────────────────────
private bool _isInKnockback;

/// <summary>Whether the entity is currently being knocked back.</summary>
public bool IsInKnockback => _isInKnockback;

// In ApplyKnockback():
_isInKnockback = true;

// In KnockbackRecovery():
_isInKnockback = false;

// ── Juggle Integration ───────────────────────────────────────

/// <summary>
/// Called by JuggleSystem when a launched entity lands after stun.
/// Triggers invulnerability blink without waiting for stun timer.
/// </summary>
public void TriggerInvulnerabilityBlink()
{
    StartCoroutine(InvulnerabilityBlink());
}

// Modify StunRoutine():
// After stun timer ends, check if airborne:
// - If JuggleSystem.IsAirborne → call juggleSystem.RequestInvulnerabilityOnLanding()
// - If grounded → run InvulnerabilityBlink() immediately (current behavior)
```

Also modify `ApplyLaunch()` to notify JuggleSystem:
```csharp
public void ApplyLaunch(Vector2 force)
{
    if (_isDead) return;
    Vector2 reducedForce = force * (1f - enemyData.knockbackResistance);
    Rb.AddForce(reducedForce, ForceMode2D.Impulse);

    // Notify JuggleSystem if present
    var juggle = GetComponent<JuggleSystem>();
    juggle?.Launch(reducedForce, /* gravityMultiplier from DamagePacket */);
}
```

**Note:** `ApplyLaunch` needs the gravity multiplier from DamagePacket. Since `TakeDamage` calls both `ApplyKnockback` and `ApplyLaunch`, we'll pass the full DamagePacket to `ApplyLaunch` (or store the multiplier before calling).

### 7. `Editor/Prefabs/TestDummyPrefabCreator.cs` (MODIFY)

Add to TestDummy prefab creation:
- `JuggleSystem` component on root, wired to sprite transform + JuggleConfig SO
- `WallBounceHandler` component on root, wired to JuggleConfig SO + EnemyBase ref
- Create `DefaultJuggleConfig` SO at `ScriptableObjects/Combat/` if missing

---

## Acceptance Criteria

- [ ] Wall bounce detection on collision with wall colliders during knockback
- [ ] Bounce reflects velocity with configurable retention factor
- [ ] Bounce deals velocity-based minor damage, no pressure fill
- [ ] Unlimited bounces per combo (no cap)
- [ ] JuggleSystem tracks Grounded/Launched/Falling states
- [ ] Simulated height with gravity (belt-scroll style, sprite offset)
- [ ] `juggleGravityMultiplier` in DamagePacket (default 1.0, Gale-ready)
- [ ] JuggleState enum with all 5 values (Downed/Recovering stubs for T036)
- [ ] Post-stun invulnerable landing: if airborne when stun ends, blink on land
- [ ] JuggleConfig SO with all tuning values exposed in Inspector
- [ ] TestDummy prefab updated with both components via Creator Script
- [ ] No cross-pillar imports — JuggleSystem stays in Combat, queries via Shared interfaces
- [ ] `IsInKnockback` flag on EnemyBase

---

## Execution Order

1. `Shared/Enums/JuggleState.cs` — enum first, no dependencies
2. `Shared/Data/DamagePacket.cs` — add `juggleGravityMultiplier` field
3. `Shared/Data/JuggleConfig.cs` — SO for tuning values
4. `Combat/JuggleSystem.cs` — airborne simulation + state tracking
5. `Combat/WallBounceHandler.cs` — wall collision + bounce physics + damage
6. `World/EnemyBase.cs` — `IsInKnockback` flag, launch→JuggleSystem wiring, stun interaction
7. `Editor/Prefabs/TestDummyPrefabCreator.cs` — wire components + create default config SO

---

## Dependencies

**Reads from (no modification):**
- `AttackData.causesWallBounce`, `AttackData.causesLaunch` — already defined
- `EnemyData.knockbackResistance` — already used in ApplyKnockback/ApplyLaunch
- `IDamageable` interface — already defined

**Modifies:**
- `DamagePacket` — new field (Shared, requires review)
- `EnemyBase` — new flag + hooks (World pillar, but IDamageable contract stays stable)

**Sets up for:**
- T036 (OTG vs TechHit) — `JuggleState.Downed`/`Recovering` values ready, JuggleSystem exposes `CurrentState`

---

## Risks & Gotchas

1. **DamagePacket is a readonly struct** — adding a field changes its constructor signature. All existing call sites that construct DamagePacket need updating. Use a default parameter (`float juggleGravityMultiplier = 1f`) to minimize churn.
2. **Wall colliders in test scene** — MovementTestSceneCreator already creates 4 invisible wall colliders. Verify they are `Collider2D` (not trigger) so `OnCollisionEnter2D` fires. May need a "Wall" layer.
3. **Belt-scroll launch vs Rigidbody Y velocity** — Launch applies impulse to Rigidbody2D Y, but JuggleSystem simulates height separately (like CharacterMotor). Need to ensure they don't conflict — JuggleSystem should zero out Rb Y velocity on launch and manage height internally.
4. **EnemyBase.ApplyLaunch needs gravity multiplier** — `TakeDamage` calls `ApplyLaunch(damage.launchForce)` but doesn't pass the gravity multiplier. Refactor to pass it through, either by changing the signature or storing the DamagePacket temporarily.
