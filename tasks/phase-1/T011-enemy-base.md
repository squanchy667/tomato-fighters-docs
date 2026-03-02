# T011: EnemyBase — IDamageable + IAttacker

## Metadata
| Field | Value |
|-------|-------|
| **Phase** | 1 — Foundation |
| **Type** | implementation |
| **Priority** | P0 |
| **Owner** | Dev 3 |
| **Agent** | world-agent |
| **Depends On** | T001 |
| **Blocks** | T013, T022 |
| **Status** | PENDING |
| **Branch** | `pillar3/T011-enemy-base` |

## Objective
Implement the base class that all enemies inherit from, fully implementing the `IDamageable` and `IAttacker` shared interfaces so that the Combat pillar can damage enemies and enemies can attack players through the shared contract layer.

## Context
`EnemyBase` is the foundational class for the entire enemy hierarchy in the World pillar. It sits at `Assets/Scripts/World/EnemyBase.cs` and bridges the gap between World-owned enemies and Combat-pillar damage/attack contracts.

The shared interfaces it must implement (defined in T001):
- **`IDamageable`**: health management, pressure meter, knockback via Rigidbody2D, launch, invulnerability frames
- **`IAttacker`**: current attack data, telegraph type (Normal/Unstoppable), punish state (whether enemy is in a punishable recovery window)

All stat values come from an `EnemyData` ScriptableObject — no hardcoded numbers.

Key architecture rules:
- No singletons
- Rigidbody2D for ALL physics (knockback, launch) — never `transform.position` manipulation
- ScriptableObjects for data (EnemyData SO defines HP, pressure threshold, knockback resistance, etc.)
- Animation Events for timing (death animations, hit reactions)

## Requirements
1. Create `EnemyBase` abstract MonoBehaviour at `Assets/Scripts/World/EnemyBase.cs`
2. Implement `IDamageable` interface fully:
   - `TakeDamage(DamagePacket packet)` — apply damage, reduce health, fill pressure meter, apply knockback/launch via Rigidbody2D
   - `CurrentHealth` and `MaxHealth` properties
   - `IsAlive` property
   - `IsInvulnerable` property — true during invulnerability frames after stun recovery
   - `PressureMeter` — current pressure value (0 to threshold)
   - `ApplyKnockback(Vector2 force)` — add force via Rigidbody2D
   - `ApplyLaunch(Vector2 force)` — upward force for air juggle via Rigidbody2D
   - Death callback/event when health reaches 0
3. Implement `IAttacker` interface fully:
   - `CurrentAttack` — returns the active `AttackData` SO (null if not attacking)
   - `TelegraphType` — returns the telegraph type of current attack (Normal/Unstoppable)
   - `IsPunishable` — true when enemy is in a punishable recovery window after a big attack
   - `IsAttacking` — true during active attack frames
4. Create `EnemyData` ScriptableObject at `Assets/Scripts/Shared/Data/EnemyData.cs` (or `Assets/Scripts/World/EnemyData.cs`):
   - Max HP, pressure threshold, knockback resistance, movement speed
   - Attack list (references to `AttackData` SOs)
   - Invulnerability duration after stun recovery
   - `[CreateAssetMenu]` attribute for easy Inspector creation
5. Read all base stats from `[SerializeField] EnemyData` reference — no hardcoded values
6. Implement pressure meter:
   - Fills when taking damage (punish damage fills 2x faster)
   - When full → enemy enters stunned state for configurable duration
   - After stun ends → invulnerable recovery period with visual blink (white sprite flash)
   - Pressure resets after stun recovery
7. Implement invulnerability after stun recovery:
   - Duration read from `EnemyData`
   - Visual feedback: white sprite blink (SpriteRenderer color flash)
   - `TakeDamage` returns early if `IsInvulnerable` is true
8. Provide virtual methods for subclass overrides: `OnDamaged()`, `OnStunned()`, `OnDeath()`, `OnRecovery()`
9. Fire death event (UnityEvent or SO channel) that `WaveManager` can subscribe to for alive-count tracking

## File Plan
| File | Purpose |
|------|---------|
| `Assets/Scripts/World/EnemyBase.cs` | Abstract base MonoBehaviour implementing IDamageable + IAttacker |
| `Assets/Scripts/Shared/Data/EnemyData.cs` | ScriptableObject defining enemy stats (HP, pressure threshold, knockback resistance, speed, attacks) |

## Implementation Notes
- `EnemyBase` should be abstract — concrete enemies (T022 EnemyAI, T042 HeavyEnemy) inherit from it
- Knockback: use `Rigidbody2D.AddForce(force, ForceMode2D.Impulse)` — never set `transform.position`
- Knockback resistance: multiply incoming knockback force by `(1 - knockbackResistance)` from `EnemyData`
- Pressure meter: `pressureDamage = packet.damage * (isPunishDamage ? 2f : 1f)` — punish state is when `IsPunishable` was true on the attacker when the hit landed
- Stun blink: use a coroutine that toggles `SpriteRenderer.color` between white and original at a fast interval (e.g., every 0.1s) during invulnerability
- Cache component references (`Rigidbody2D`, `SpriteRenderer`, `Animator`) in `Awake()` using `GetComponent<T>()`
- Death: disable colliders, fire death event, play death animation (via `Animator.SetTrigger`), then `Destroy(gameObject)` after animation completes (or pool return)
- The `IAttacker` implementation will be minimal in Phase 1 — `CurrentAttack` returns null, `IsAttacking` returns false, `IsPunishable` returns false. T022 (BasicEnemyAI) adds actual attack behavior
- Do NOT reference any Combat or Roguelite pillar code — only use Shared interfaces, data, and events

## Acceptance Criteria
- [ ] `EnemyBase` implements `IDamageable` fully — all interface members have working implementations
- [ ] `EnemyBase` implements `IAttacker` fully — all interface members have working implementations (stubs for Phase 1)
- [ ] Health system works: takes damage, tracks current/max HP, fires death event at 0
- [ ] Pressure meter fills on damage, triggers stun at threshold, resets after recovery
- [ ] Knockback applied via `Rigidbody2D.AddForce` — never transform manipulation
- [ ] Invulnerability after stun recovery with visual white blink on `SpriteRenderer`
- [ ] All stats read from `EnemyData` ScriptableObject — no hardcoded values
- [ ] `EnemyData` SO has `[CreateAssetMenu]` attribute and all required fields
- [ ] Virtual hook methods (`OnDamaged`, `OnStunned`, `OnDeath`, `OnRecovery`) for subclass extension
- [ ] Death event fires so `WaveManager` can track alive count
- [ ] No references to Combat or Roguelite pillar code
- [ ] Compiles with zero warnings

## References
- [System Overview](../../architecture/system-overview.md) — IDamageable (P1 -> P3), IAttacker (P3 -> P1) interface flow
- [Data Flow](../../architecture/data-flow.md) — Step 6: IDamageable.TakeDamage(DamagePacket)
- [TASK_BOARD.md](../../TASK_BOARD.md) — T011 entry
- [PLAN.md](../../PLAN.md) — Phase 1 scope, interface-only coupling
