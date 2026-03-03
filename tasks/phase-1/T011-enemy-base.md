# T011: EnemyBase — IDamageable + IAttacker

## Metadata
| Field | Value |
|-------|-------|
| **Phase** | 1 — Foundation |
| **Type** | implementation |
| **Priority** | P0 |
| **Owner** | Dev 1 (building for Dev 3 handoff) |
| **Agent** | world-agent |
| **Depends On** | T001 |
| **Blocks** | T013, T022 |
| **Status** | DONE |
| **Completed** | 2026-03-03 |
| **Branch** | `pillar3/T011-enemy-base` |

## Objective
Implement the base class that all enemies inherit from, fully implementing the `IDamageable` and `IAttacker` shared interfaces so that the Combat pillar can damage enemies and enemies can attack players through the shared contract layer. Also provides a concrete `TestDummyEnemy` for end-to-end combat testing, and a stub `PlayerDamageable` so the player can receive hits.

## Context
`EnemyBase` is the foundational class for the entire enemy hierarchy in the World pillar. It sits at `Assets/Scripts/World/EnemyBase.cs` and bridges the gap between World-owned enemies and Combat-pillar damage/attack contracts.

The shared interfaces it must implement (defined in T001):
- **`IDamageable`**: health management, pressure meter, knockback via Rigidbody2D, launch, invulnerability frames
- **`IAttacker`**: current attack data, telegraph type (Normal/Unstoppable), punish state (whether enemy is in a punishable recovery window)

All stat values come from an `EnemyData` ScriptableObject — no hardcoded numbers.

**Why Dev 1 is doing this:** Dev 3 is focused on animations. Dev 1 (Combat) needs a target to validate the hitbox pipeline (T015) end-to-end and to test the upcoming DefenseSystem (T016). Dev 3 inherits the clean base class and builds T022 (BasicEnemyAI) on top.

Key architecture rules:
- No singletons
- Rigidbody2D for ALL physics (knockback, launch) — never `transform.position` manipulation
- ScriptableObjects for data (EnemyData SO defines HP, pressure threshold, knockback resistance, etc.)
- Animation Events for timing (death animations, hit reactions)

## Design Decisions

### DD-1: EnemyData lives in `World/EnemyData.cs`
**Rationale:** Keep it in Dev 3's pillar domain. Dev 1 and Dev 3 are in high sync and will do a direct handoff. If Combat ever needs enemy stats, we'll revisit through a Shared interface.

### DD-2: Attack timer in TestDummyEnemy subclass, not EnemyBase
**Rationale:** EnemyBase stays abstract and clean. The attack timer is debug/test logic that doesn't belong in the base class. Dev 3's T022 AI state machine replaces TestDummyEnemy naturally. Keeps the inheritance clean:
```
EnemyBase (abstract)
├── TestDummyEnemy   ← timer-based, for Dev 1 testing
├── BasicEnemyAI     ← T022, Dev 3 builds state machine
└── HeavyEnemy       ← T042, future
```

### DD-3: HitboxDamage moves to Shared
**Rationale:** `HitboxDamage` has zero Combat-pillar dependencies (only uses `System`, `UnityEngine`, `TomatoFighters.Shared.Interfaces`). The trigger detection logic is identical for player and enemy hitboxes. Moving it to Shared avoids code duplication and lets both Combat's `HitboxManager` and World's `TestDummyEnemy` reference the same component.

### DD-4: HitboxDamage location: `Shared/Components/HitboxDamage.cs`
**Rationale:** It's a reusable MonoBehaviour, not an interface or data struct. `Shared/Components/` distinguishes it from `Shared/Interfaces/`, `Shared/Data/`, and `Shared/Enums/`.

### DD-5: Stub PlayerDamageable in Combat pillar
**Rationale:** The player currently can hit enemies but can't be hit back. Without `IDamageable` on the player, the dummy's attacks collide but `GetComponentInParent<IDamageable>()` finds nothing. A minimal stub (log damage + sprite flash) is enough to:
- Verify the full bidirectional damage pipeline
- Provide a target for T016 (DefenseSystem) testing
- Not a full PlayerHealth system — that's future scope

## Requirements
1. **Move `HitboxDamage`** from `Combat/Hitbox/` to `Shared/Components/`
   - Update namespace from `TomatoFighters.Combat` to `TomatoFighters.Shared.Components`
   - Update `HitboxManager.cs` import to reference new location
   - Update Shared asmdef if needed (should already work since HitboxDamage only uses Shared types)

2. **Create `EnemyData` ScriptableObject** at `Assets/Scripts/World/EnemyData.cs`:
   - Max HP, pressure threshold, knockback resistance, movement speed
   - Attack list (references to `AttackData` SOs)
   - Stun duration, invulnerability duration after stun recovery
   - `[CreateAssetMenu]` attribute for easy Inspector creation

3. **Create `EnemyBase` abstract MonoBehaviour** at `Assets/Scripts/World/EnemyBase.cs`:
   - Implement `IDamageable` fully:
     - `TakeDamage(DamagePacket)` — apply damage, fill pressure, apply knockback/launch
     - `CurrentHealth` / `MaxHealth` properties
     - `AddStun(float)` — fill pressure meter
     - `IsStunned` — true during stun state
     - `ApplyKnockback(Vector2)` / `ApplyLaunch(Vector2)` — via Rigidbody2D.AddForce
     - `IsInvulnerable` — true during post-stun recovery blink
   - Implement `IAttacker` as virtual stubs:
     - `CurrentAttack` → null (T022 overrides)
     - `IsCurrentAttackUnstoppable` → false
     - `CurrentTelegraphType` → Normal
     - `PunishWindowDuration` → 0f
     - `IsInPunishableState` → false
   - Pressure meter: fills on damage, punish = 2x rate, stun at threshold, reset on recovery
   - Invulnerability: post-stun blink via coroutine (SpriteRenderer.color toggle)
   - Virtual hooks: `OnDamaged()`, `OnStunned()`, `OnDeath()`, `OnRecovery()`
   - Death event: `System.Action OnDied` for WaveManager subscription
   - Cache Rigidbody2D, SpriteRenderer, Collider2D[] in Awake()

4. **Create `TestDummyEnemy`** at `Assets/Scripts/World/TestDummyEnemy.cs`:
   - Inherits `EnemyBase`
   - `[SerializeField] float attackInterval` — configurable timer
   - `[SerializeField] AttackData attackData` — what attack to use
   - On timer: activates its hitbox child, sets `CurrentAttack`, deactivates after active frames
   - Overrides IAttacker virtuals to return real values during attack window
   - Uses `HitboxDamage` component on child hitbox (from Shared)
   - Builds its own `DamagePacket` on hit and applies to target

5. **Create `PlayerDamageable` stub** at `Assets/Scripts/Combat/PlayerDamageable.cs`:
   - Implements `IDamageable`
   - Logs damage to console
   - Sprite flash (red tint) on hit for visual feedback
   - Minimal health tracking (100 HP, no death logic yet)
   - Knockback/launch via player's Rigidbody2D

## File Plan
| File | Action | Location | Purpose |
|------|--------|----------|---------|
| `HitboxDamage.cs` | **Move** | `Shared/Components/` | Shared trigger detection for both pillars |
| `EnemyData.cs` | **New** | `World/` | SO: enemy stats (HP, pressure, knockback resist, attacks) |
| `EnemyBase.cs` | **New** | `World/` | Abstract base: IDamageable + IAttacker (virtual stubs) |
| `TestDummyEnemy.cs` | **New** | `World/` | Concrete test enemy: timer attack, hitbox, hurtbox |
| `PlayerDamageable.cs` | **New** | `Combat/` | Stub IDamageable on player (log + flash) |
| `HitboxManager.cs` | **Update** | `Combat/Hitbox/` | Fix HitboxDamage import path |

## Execution Order
1. Move `HitboxDamage` to `Shared/Components/`, update imports in `HitboxManager`
2. `EnemyData.cs` — SO definition (no dependencies)
3. `EnemyBase.cs` — abstract base implementing both interfaces
4. `TestDummyEnemy.cs` — concrete subclass with timer attack + hitbox
5. `PlayerDamageable.cs` — stub on player for bidirectional testing
6. Create 1x `EnemyData` asset + 1x `AttackData` asset for the dummy's attack
7. Verify compilation, test in scene

## Implementation Notes
- `EnemyBase` should be abstract — concrete enemies (T022 EnemyAI, T042 HeavyEnemy) inherit from it
- Knockback: use `Rigidbody2D.AddForce(force, ForceMode2D.Impulse)` — never set `transform.position`
- Knockback resistance: multiply incoming knockback force by `(1 - knockbackResistance)` from `EnemyData`
- Pressure meter: `pressureFill = packet.amount * (packet.isPunishDamage ? 2f : 1f)`
- Stun blink: coroutine that toggles `SpriteRenderer.color` between white and original every 0.1s during invulnerability
- Cache component references (`Rigidbody2D`, `SpriteRenderer`) in `Awake()` using `GetComponent<T>()`
- Death: disable colliders, fire `OnDied` event, play death animation (via `Animator.SetTrigger` if Animator present), then `Destroy(gameObject)` after delay (or pool return)
- TestDummyEnemy: no Animator needed — just timer → activate hitbox child → deactivate after duration
- PlayerDamageable: no death state, no respawn — just logs and flashes for now
- Do NOT reference any Roguelite pillar code — World references Shared only, Combat references Shared only

## Acceptance Criteria
- [ ] `HitboxDamage` moved to `Shared/Components/` with updated namespace
- [ ] `HitboxManager` compiles with new import path
- [ ] `EnemyBase` implements `IDamageable` fully — all interface members have working implementations
- [ ] `EnemyBase` implements `IAttacker` fully — virtual stubs, overridable by subclasses
- [ ] Health system works: takes damage, tracks current/max HP, fires death event at 0
- [ ] Pressure meter fills on damage, triggers stun at threshold, resets after recovery
- [ ] Knockback applied via `Rigidbody2D.AddForce` — never transform manipulation
- [ ] Invulnerability after stun recovery with visual white blink on `SpriteRenderer`
- [ ] All stats read from `EnemyData` ScriptableObject — no hardcoded values
- [ ] `EnemyData` SO has `[CreateAssetMenu]` attribute and all required fields
- [ ] Virtual hook methods (`OnDamaged`, `OnStunned`, `OnDeath`, `OnRecovery`) for subclass extension
- [ ] Death event fires so `WaveManager` can track alive count
- [ ] `TestDummyEnemy` attacks on configurable timer interval
- [ ] `TestDummyEnemy` hitbox on `EnemyHitbox` layer, hurtbox on `EnemyHurtbox` layer
- [ ] `PlayerDamageable` stub receives damage, logs, and flashes
- [ ] Bidirectional damage pipeline works: player → enemy AND enemy → player
- [ ] No cross-pillar import violations (World→Shared only, Combat→Shared only)
- [ ] Compiles with zero warnings

## Prefab Structure

### TestDummyEnemy Prefab
```
TestDummy (root)
├── EnemyBase components: Rigidbody2D (gravity=0), SpriteRenderer
├── TestDummyEnemy component
├── BoxCollider2D (body, EnemyHurtbox layer) — can be hit by player
└── Hitbox_Punch (child)
    ├── BoxCollider2D (trigger, EnemyHitbox layer) — hits player
    ├── HitboxDamage component (from Shared)
    └── Starts disabled, activated by timer
```

### Player Additions
```
Player (existing prefab)
├── ... existing components ...
├── PlayerDamageable component (NEW)
└── BoxCollider2D (PlayerHurtbox layer) (NEW or existing body collider re-layered)
```

## Layer Collision Matrix (verify)
- `PlayerHitbox` ↔ `EnemyHurtbox` = yes (player attacks hit enemy)
- `EnemyHitbox` ↔ `PlayerHurtbox` = yes (enemy attacks hit player)
- All other hitbox/hurtbox combos = no

## References
- [System Overview](../../architecture/system-overview.md) — IDamageable (P1 → P3), IAttacker (P3 → P1) interface flow
- [Data Flow](../../architecture/data-flow.md) — Step 6: IDamageable.TakeDamage(DamagePacket)
- [TASK_BOARD.md](../../TASK_BOARD.md) — T011 entry
- [PLAN.md](../../PLAN.md) — Phase 1 scope, interface-only coupling
