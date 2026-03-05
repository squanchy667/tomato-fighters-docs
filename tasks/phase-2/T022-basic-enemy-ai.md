# T022: BasicEnemyAI

> **Phase:** 2 | **Priority:** P0 | **Owner:** Dev 3 (World) | **Depends on:** T011
> **Blocks:** T023, T032, T042

---

## Summary

6-state enemy AI state machine: Idle -> Patrol -> Chase -> Attack -> HitReact -> Death. Plain C# state classes driven by `EnemyAI` MonoBehaviour via `Tick(dt)`. AI behavior values live on `EnemyData` SO. Player targeting uses Physics2D overlap (layer-based) with `ForceTarget()` hook for future Provoke/taunt. Creates a concrete `BasicMeleeEnemy` for testing.

---

## Design Decisions

### DD-1: State Machine Architecture — Hybrid (Tick + Coroutine)

Plain C# state machine for the overall structure. `EnemyAI.Update()` calls `currentState.Tick(dt)`. States are pure C# classes inheriting from `EnemyStateBase` with `Enter()`, `Tick(dt)`, `Exit()` lifecycle. `AttackState` internally uses a coroutine (started via `EnemyAI.StartCoroutine()`) for the telegraph -> hitbox -> cooldown timed sequence.

**Rationale:** Matches `ComboStateMachine` / `MovementStateMachine` pattern. Tick-based is testable; coroutine is cleaner for linear timed sequences.

### DD-2: AI Config — Fields on EnemyData SO (Option A)

AI behavior fields (aggroRange, attackRange, patrolRadius, etc.) added directly to the existing `EnemyData` ScriptableObject. No separate `EnemyAIConfig` SO.

**Rationale:** `EnemyData` already has `movementSpeed` and `attacks[]`. One SO per enemy type, everything in one place. Every enemy type needs unique stats AND unique behavior — no practical reuse case for splitting.

```csharp
// Added to EnemyData.cs
[Header("AI Behavior")]
public float aggroRange = 8f;
public float attackRange = 1.5f;
public float patrolRadius = 3f;
public float leashRange = 12f;
public float idleDuration = 2f;
public float attackCooldown = 1.5f;
[Range(0f, 1f)]
public float aggression = 0.5f;
```

### DD-3: Player Targeting — Physics2D Overlap + ForceTarget

Target selection uses `Physics2D.OverlapCircleAll()` on the Player layer. Picks nearest player. Periodic checks (every 0.5s or on state enter), not every frame. `ForceTarget(Transform target, float duration)` method allows Provoke (T028) to override targeting.

**Rationale:** Future-proofs for co-op (T051). No singletons, no FindObjectOfType. Force target cleanly separates taunt mechanics from base AI.

```csharp
public void ForceTarget(Transform target, float duration)
{
    forcedTarget = target;
    forcedTargetExpiry = Time.time + duration;
}

public Transform CurrentTarget => forcedTarget != null ? forcedTarget : currentTarget;
```

When taunt expires, position-based targeting resumes automatically on next `UpdateTarget()` call.

### DD-4: HitReact Interrupts — Unstoppable = Super Armor

- Normal attacks: enemy transitions to HitReact when hit
- Unstoppable attacks (`TelegraphType.Unstoppable`): enemy ignores hit reactions (super armor)
- Stun always works: pressure meter fills regardless of attack type, stun overrides everything

**Rationale:** Simple binary rule. T023 can add nuance (poise thresholds) later if needed.

### DD-5: TestDummyEnemy — Leave As-Is

TestDummyEnemy is not refactored. It remains a simple debug dummy with timer-based attacks. `BasicMeleeEnemy` is the first real user of the EnemyAI state machine.

**Rationale:** TestDummyEnemy is intentionally simple for combat debugging. Separate concerns.

### DD-6: Create BasicMeleeEnemy in This Task

A concrete `BasicMeleeEnemy` class extending `EnemyBase` with `EnemyAI` attached. Includes a Creator Script for the prefab and an EnemyData SO. Used for testing the AI state machine end-to-end.

---

## File Plan

| # | File | Pillar | Purpose |
|---|------|--------|---------|
| 1 | `Scripts/World/EnemyStateBase.cs` | World | Abstract base class: `Enter()`, `Tick(dt)`, `Exit()`. Holds `EnemyAI` context reference. |
| 2 | `Scripts/World/EnemyAI.cs` | World | MonoBehaviour. Owns state machine, ticks current state. Caches EnemyBase, Rigidbody2D, player layer. Exposes `TransitionTo()`, `ForceTarget()`, `UpdateTarget()`, `StartCoroutine()`. Subscribes to EnemyBase hooks for HitReact/Death. |
| 3 | `Scripts/World/States/IdleState.cs` | World | Wait idleDuration, then Patrol. Transitions to Chase if player in aggro range. |
| 4 | `Scripts/World/States/PatrolState.cs` | World | Pick random point within patrolRadius, move via Rigidbody2D.MovePosition. On arrival -> Idle. Chase if player in aggro range. |
| 5 | `Scripts/World/States/ChaseState.cs` | World | Move toward CurrentTarget. Attack when in attackRange. Return to Patrol if target beyond leashRange. Periodic target updates. |
| 6 | `Scripts/World/States/AttackState.cs` | World | Pick attack from EnemyData.attacks[]. Coroutine: telegraph -> clash window -> activate hitbox -> deal damage -> cooldown. Transitions to Chase after cooldown. |
| 7 | `Scripts/World/States/HitReactState.cs` | World | Entered on EnemyBase.OnDamaged (if not Unstoppable). Stagger duration then -> Chase. If stunned, stays for full stun duration, transitions on OnRecovery. |
| 8 | `Scripts/World/States/DeathState.cs` | World | Entered on EnemyBase.OnDeath. Disables colliders, terminal state. |
| 9 | `Scripts/World/BasicMeleeEnemy.cs` | World | Concrete enemy: extends EnemyBase, requires EnemyAI. Minimal — just wires the two together. |
| 10 | `Scripts/World/EnemyData.cs` | World | **Modified** — add AI behavior fields (aggroRange, attackRange, patrolRadius, leashRange, idleDuration, attackCooldown, aggression). |
| 11 | `Editor/Prefabs/BasicEnemyPrefabCreator.cs` | Editor | Creator Script for BasicMeleeEnemy prefab + EnemyData SO with tuned values. |

---

## Imports (all allowed)

All World files import only from:
- `TomatoFighters.Shared.Interfaces` (IDamageable, IAttacker, IDefenseProvider)
- `TomatoFighters.Shared.Data` (AttackData, DamagePacket, EnemyData)
- `TomatoFighters.Shared.Enums` (TelegraphType, DamageResponse, DamageType)
- `TomatoFighters.Shared.Components` (HitboxDamage, ClashTracker)
- `TomatoFighters.Shared.Events` (VoidEventChannel)
- `System`, `System.Collections`, `UnityEngine`

No cross-pillar violations.

---

## Acceptance Criteria

- [ ] 6-state machine with clean transitions (Idle/Patrol/Chase/Attack/HitReact/Death)
- [ ] Configurable per-enemy via EnemyData SO: aggression, frequency, range
- [ ] Uses AttackData for attack patterns
- [ ] Telegraph visuals (normal vs red/unstoppable)
- [ ] Inherits from EnemyBase
- [ ] Physics2D player detection (no singletons, no FindObjectOfType)
- [ ] ForceTarget() hook for future Provoke integration
- [ ] Normal attacks interruptible, Unstoppable = super armor
- [ ] BasicMeleeEnemy prefab via Creator Script
- [ ] Pillar boundaries clean (World + Shared only)

---

## Execution Order

1. `EnemyStateBase.cs` (no dependencies)
2. `EnemyData.cs` (add AI fields)
3. `EnemyAI.cs` (shell — depends on EnemyStateBase + EnemyData)
4. `IdleState.cs` + `PatrolState.cs` (simple movement)
5. `ChaseState.cs` (needs player detection)
6. `AttackState.cs` (most complex — telegraph + hitbox)
7. `HitReactState.cs` + `DeathState.cs` (react to EnemyBase events)
8. `BasicMeleeEnemy.cs` (concrete enemy)
9. `BasicEnemyPrefabCreator.cs` (Creator Script)
