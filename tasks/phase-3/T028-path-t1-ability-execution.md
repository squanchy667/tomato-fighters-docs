# T028: Path T1 Ability Execution

> **Phase:** 3 | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T017, T018
> **Status:** DONE | **Completed:** 2026-03-05 | **Branch:** `combat/T028-path-t1-ability-execution`

## Summary

Implement the `PathAbilityExecutor` framework and all 12 Tier 1 path abilities. The executor lives on the player, queries `IPathProvider` for unlocked abilities, routes input, manages cooldowns/mana, and fires `PathAbilityEventData` through `ICombatEvents`. Each ability reads from `PathData` SO and checks tier before executing.

## Acceptance Criteria

- [x] All 12 T1 abilities functional
- [x] Cooldown management per ability
- [x] Mana consumption where specified
- [x] Integrates with IPathProvider tier checks
- [x] Fires PathAbilityEventData through ICombatEvents

## Design Decisions

### DD-1: Two Ability Input Buttons
**Decision:** Two dedicated input buttons — Q for Main path ability, E for Secondary path ability.
**Rationale:** Players can have up to 2 T1 abilities active (Main + Secondary). Two buttons provide clear, unambiguous mapping. Standard in action games with skill systems.

```csharp
// Add to InputActionAsset:
// Ability1 = Q (Main path ability)
// Ability2 = E (Secondary path ability)

// CharacterInputHandler routes:
// Ability1 → PathAbilityExecutor.ActivateMainAbility()
// Ability2 → PathAbilityExecutor.ActivateSecondaryAbility()
```

### DD-2: Framework + Core Combat, Stub Ally Abilities
**Decision:** Full implementation for 9 abilities that work in solo. Ally-targeting abilities (ShieldLink, MendingAura, Empower) get self-target fallback.
**Rationale:** Co-op (T051) and AI companions (T052) don't exist yet. Building full ally-targeting infrastructure for 3 abilities is premature. Self-targeting keeps them functional and testable.

| Ability | Solo Fallback |
|---------|--------------|
| ShieldLink | No effect (can't redirect damage to self meaningfully) |
| MendingAura | Heals self only |
| Empower | Buffs self |

### DD-3: Mana via Existing PlayerManaTracker
**Decision:** Use the existing `PlayerManaTracker` component (built in T025) via `[SerializeField]` injection.
**Rationale:** Already implements `TryConsume(float)`, `Restore(float)`, auto-regen, and HUD event firing. No new mana system needed.

```csharp
// In PathAbilityExecutor:
[SerializeField] private PlayerManaTracker manaTracker;

// In ability activation:
if (!manaTracker.TryConsume(ability.ManaCost)) return; // Not enough mana
```

### DD-4: Passive Modifiers via Interface Query in HitboxManager
**Decision:** HitboxManager queries an `IPathAbilityModifier` interface during hit resolution to apply CleavingStrikes/PiercingShots modifications.
**Rationale:** Clean integration point that doesn't scatter ability logic. Sets up the pattern for T2/T3 abilities that also modify attacks. HitboxManager already has the hit resolution pipeline.

```csharp
// New interface in Shared/Interfaces/:
public interface IPathAbilityModifier
{
    int GetAdditionalTargetCount();           // CleavingStrikes: 2 extra targets
    float GetAdditionalTargetDamageScale();   // CleavingStrikes: 0.6x for extra targets
    bool DoProjectilesPierce();               // PiercingShots: true
    float GetPierceDamageFalloff();           // PiercingShots: 0.8x per pierce
}
```

### DD-5: PhaseDash Modifies CharacterMotor Dash Config
**Decision:** PhaseDash modifies the existing dash system (charge count, adds damage callback) rather than replacing it entirely.
**Rationale:** CharacterMotor already has full dash infrastructure (i-frames, cooldown, animation). PhaseDash just needs to change charge count from 1→2, extend i-frames by 50%, and subscribe to dash events for pass-through damage.

```csharp
// PhaseDash modifies motor config on activation:
motor.SetDashCharges(2);
motor.SetDashIFrameMultiplier(1.5f);
// Subscribes to motor.Dashed event for damage application
```

### DD-6: Self-Target Fallback for Ally Abilities
**Decision:** ShieldLink has no effect in solo, MendingAura heals self, Empower buffs self.
**Rationale:** Simplest approach. These abilities will get proper ally-targeting when co-op (T051) or companions (T052) are implemented. Self-targeting validates the cooldown/mana/event pipeline.

### DD-7: SummonSproutling Reuses Enemy Framework Inverted
**Decision:** Create a `SproutlingEnemy` that derives from `EnemyBase` + uses `EnemyAI` but targets the enemy layer instead of the player layer.
**Rationale:** The existing enemy framework already handles health, hit detection, state machine, and targeting via `Physics2D.OverlapCircleAll` with configurable layer masks. Reusing it avoids building a separate companion AI system and validates the framework's flexibility.

```csharp
// SproutlingEnemy : EnemyBase
// - HP: 40, ATK: 0.6, lifetime: 20s
// - EnemyAI targeting layer = EnemyHurtbox (attacks other enemies)
// - Max 2 active, tracked by SummonSproutling ability
// - Inherits 30% of player's ritual effects (future: via IBuffProvider query)
```

### DD-8: Lightweight ProjectileBase in Shared
**Decision:** Create a reusable `ProjectileBase` class in `Shared/Components/` that handles travel, collision, and lifetime. `HarpoonProjectile` subclass adds immobilize effect.
**Rationale:** Viper's entire kit is ranged, and T2/T3 abilities (Rapid Fire, Killshot, Mana Blast, Chain Slash) will all need projectiles. A small base class now prevents rework later.

```csharp
// Shared/Components/ProjectileBase.cs
public abstract class ProjectileBase : MonoBehaviour
{
    [SerializeField] protected float speed = 15f;
    [SerializeField] protected float lifetime = 3f;
    [SerializeField] protected LayerMask targetLayer;

    protected Rigidbody2D rb;
    protected float timer;

    protected virtual void OnTargetHit(IDamageable target, Collider2D collider) { }
    protected virtual void OnLifetimeExpired() => Destroy(gameObject);
}
```

### DD-9: StatusEffectTracker Component on Enemies
**Decision:** A lightweight `StatusEffectTracker` component on enemies that holds active effects. Abilities add effects, tracker ticks durations, systems query it.
**Rationale:** Centralizes effect tracking instead of scattering duration management across 4+ abilities. Small component (~60-80 lines) with big payoff.

```csharp
// StatusEffect struct:
public struct StatusEffect
{
    public StatusEffectType type;  // Mark, Immobilize, Slow, Taunt
    public float duration;
    public float magnitude;        // Slow: 0.3 = 30% slow; Mark: 0.25 = 25% bonus damage
    public Transform source;       // Who applied it
}

// On enemies:
public class StatusEffectTracker : MonoBehaviour, IStatusEffectable
{
    public void AddEffect(StatusEffect effect);
    public bool HasEffect(StatusEffectType type);
    public StatusEffect? GetEffect(StatusEffectType type);
    public float GetSlowMultiplier();  // 1.0 = no slow
    public bool IsImmobilized();
}
```

### DD-10: Cross-Pillar Contracts via Shared Interfaces
**Decision:** `IStatusEffectable` interface and `ProjectileBase` go in `Shared/`. `StatusEffectTracker` implements `IStatusEffectable` in World pillar. Combat abilities reference only the shared interface.
**Rationale:** Non-negotiable per project architecture — pillar boundaries must not be crossed. Abilities in Combat call `IStatusEffectable.AddEffect()`, World's `StatusEffectTracker` handles the implementation.

```
Shared/Interfaces/IStatusEffectable.cs      ← Combat abilities call this
Shared/Interfaces/IPathAbilityModifier.cs   ← HitboxManager queries this
Shared/Components/ProjectileBase.cs         ← Both pillars can use
Shared/Enums/StatusEffectType.cs            ← Shared enum
Shared/Data/StatusEffect.cs                 ← Shared struct
World/StatusEffectTracker.cs                ← Implements IStatusEffectable
```

## File Plan

### Shared Layer (new files)

| File | Purpose |
|------|---------|
| `Shared/Interfaces/IPathAbility.cs` | Common ability interface: `AbilityId`, `ActivationType`, `TryActivate()`, `Tick()`, `Cleanup()` |
| `Shared/Interfaces/IPathAbilityModifier.cs` | Query interface for passive combat modifiers (multi-target, pierce) |
| `Shared/Interfaces/IStatusEffectable.cs` | Apply/query status effects on entities |
| `Shared/Enums/StatusEffectType.cs` | `Mark`, `Immobilize`, `Slow`, `Taunt` |
| `Shared/Enums/AbilityActivationType.cs` | `Active`, `Toggle`, `Channeled`, `Passive` |
| `Shared/Data/StatusEffect.cs` | Struct: type, duration, magnitude, source |
| `Shared/Components/ProjectileBase.cs` | Reusable projectile: travel, collision, lifetime |

### Combat Pillar — Framework

| File | Purpose |
|------|---------|
| `Characters/PathAbilityExecutor.cs` | MonoBehaviour on player. Creates abilities on path selection, routes input (Q/E), manages cooldowns, fires `PathAbilityEventData` |
| `Characters/Abilities/PathAbilityContext.cs` | Dependency bundle: motor, combo, hitbox, mana, events, transform, IPathProvider |

### Combat Pillar — 12 Ability Implementations

| File | Category | Key Behavior |
|------|----------|-------------|
| `Characters/Abilities/Warden/Provoke.cs` | Active | AoE taunt via `IStatusEffectable`, 4s, CD 8s |
| `Characters/Abilities/Bulwark/IronGuard.cs` | Toggle | Halve movement via motor, 50% DR, deflect heals 5% HP |
| `Characters/Abilities/Guardian/ShieldLink.cs` | Active | Self-target fallback (no-op or minor DR), CD 12s |
| `Characters/Abilities/Executioner/MarkForDeath.cs` | Active | Apply Mark via `IStatusEffectable`, +25% damage, 8s, CD 10s |
| `Characters/Abilities/Reaper/CleavingStrikes.cs` | Passive | Implements `IPathAbilityModifier`: +2 targets at 60% damage |
| `Characters/Abilities/Shadow/PhaseDash.cs` | Passive+ | Modifies motor: 2 charges, +50% i-frames, damage on pass-through |
| `Characters/Abilities/Sage/MendingAura.cs` | Toggle | Drains 3 MNA/s, heals self 3% HP/s (self-target fallback) |
| `Characters/Abilities/Enchanter/Empower.cs` | Active | Self-buff +30% ATK +20% SPD, 8s, CD 10s, 25 MNA |
| `Characters/Abilities/Conjurer/SummonSproutling.cs` | Active | Spawn `SproutlingEnemy`, max 2, CD 12s, 30 MNA |
| `Characters/Abilities/Marksman/PiercingShots.cs` | Passive | Implements `IPathAbilityModifier`: projectiles pierce, -20%/target |
| `Characters/Abilities/Trapper/HarpoonShot.cs` | Active | Spawn `HarpoonProjectile`, immobilize 2s, CD 6s, 15 MNA |
| `Characters/Abilities/Arcanist/ManaCharge.cs` | Channeled | Hold to fill charge meter (0-100%), stationary + 30% DR |

### World Pillar

| File | Purpose |
|------|---------|
| `World/StatusEffectTracker.cs` | Implements `IStatusEffectable`, ticks durations, provides queries |
| `World/SproutlingEnemy.cs` | Derives from `EnemyBase`, targets enemy layer, 20s lifetime |

### Projectile

| File | Purpose |
|------|---------|
| `Combat/Projectiles/HarpoonProjectile.cs` | Extends `ProjectileBase`, applies immobilize via `IStatusEffectable` |

### Editor (Creator Script updates)

| File | Change |
|------|--------|
| `Editor/Prefabs/PlayerPrefabCreator.cs` | Add `PathAbilityExecutor` component, wire `PlayerManaTracker` + `IPathProvider` refs |
| `Editor/Prefabs/BasicEnemyPrefabCreator.cs` | Add `StatusEffectTracker` component to enemy prefab |
| New: `Editor/Prefabs/SproutlingPrefabCreator.cs` | Creates `SproutlingEnemy` prefab + `Sproutling_EnemyData` SO |

### Input

| File | Change |
|------|--------|
| `InputSystem_Actions.inputactions` | Add `Ability1` (Q) and `Ability2` (E) actions |
| `Characters/CharacterInputHandler.cs` | Route Ability1/Ability2 to `PathAbilityExecutor` |

## Execution Order

1. Shared enums + data structs (`StatusEffectType`, `AbilityActivationType`, `StatusEffect`)
2. Shared interfaces (`IPathAbility`, `IPathAbilityModifier`, `IStatusEffectable`)
3. `ProjectileBase` in Shared
4. `StatusEffectTracker` in World + wire onto enemy prefab via creator script
5. `PathAbilityContext` + `PathAbilityExecutor` framework
6. Input actions (Q/E) + `CharacterInputHandler` routing
7. Passive modifiers: `CleavingStrikes`, `PiercingShots` + `HitboxManager` integration
8. Dash override: `PhaseDash` + `CharacterMotor` modifications
9. Active single-target: `Provoke`, `MarkForDeath`
10. Toggles: `IronGuard`, `MendingAura`
11. Mana-cost actives: `HarpoonShot` + `HarpoonProjectile`, `Empower`, `SummonSproutling` + `SproutlingEnemy`
12. Channeled: `ManaCharge`
13. Stub: `ShieldLink` (self-target fallback)
14. Creator script updates (`PlayerPrefabCreator`, `SproutlingPrefabCreator`)
15. Debug testing setup (force-unlock abilities without path selection flow)

## Dependencies

- **Requires (done):** T017 (PassiveAbilitySystem pattern), T018 (PathSystem + IPathProvider)
- **Uses:** `PlayerManaTracker` (T025), `EnemyBase`/`EnemyAI` (T011/T022), `HitboxManager` (T015), `CharacterMotor` (T002)
- **Blocked by this:** T033 (Ability VFX), T037 (T2 Passive Enhancements + Character Ultimates)
