# T023: Enemy Attack Patterns + Telegraphs

> **Phase:** 2 | **Priority:** P1 | **Owner:** Dev 3 (World) | **Depends on:** T022, T005
> **Blocks:** T032 (Boss AI), T042 (2nd Enemy Type + Mini-Bosses)

---

## Summary

Attack pattern system using AttackData sequences. Introduces `EnemyAttackPattern` ScriptableObject — a named, flat sequence of 1–3 AttackData steps with selection conditions (range, weight, cooldown). Dedicated `TelegraphVisualController` component replaces inline color tinting in AttackState with distinct Normal (yellow ramp) vs Unstoppable (red flash overlay) visuals. Creates 2–3 concrete patterns for BasicMeleeEnemy.

---

## Design Decisions

### DD-1: Patterns Alongside `attacks[]` (Option B — Fallback)

`EnemyData` gets a new `EnemyAttackPattern[] attackPatterns` field **alongside** the existing `AttackData[] attacks`. If `attackPatterns` is populated, `AttackState` uses pattern-based execution. Otherwise, falls back to picking a random single attack from `attacks[]`.

**Rationale:** Avoids breaking `TestDummyEnemy` and any existing enemies that use `attacks[]` directly. Pattern-less enemies keep working. New enemies get patterns. Incremental adoption.

### DD-2: Flat Sequence (No Conditional Branching)

`EnemyAttackPattern` is a flat ordered list of `AttackPatternStep`. Steps execute in order — no branching based on hit/miss or range mid-pattern.

**Rationale:** Beat 'em up enemies typically have fixed combo strings. Branching is overkill for Phase 2. T032 (Boss AI) can extend the pattern SO with optional branch fields if needed.

```csharp
[CreateAssetMenu(menuName = "TomatoFighters/Data/EnemyAttackPattern")]
public class EnemyAttackPattern : ScriptableObject
{
    [Header("Identity")]
    public string patternName;

    [Header("Steps")]
    public AttackPatternStep[] steps;

    [Header("Selection")]
    [Tooltip("Higher weight = more likely to be chosen")]
    public float selectionWeight = 1f;
    [Tooltip("Min distance to target for this pattern to be valid")]
    public float minRange;
    [Tooltip("Max distance to target for this pattern to be valid")]
    public float maxRange = 99f;
    [Tooltip("Seconds before this pattern can be selected again")]
    public float patternCooldown;
}

[Serializable]
public struct AttackPatternStep
{
    public AttackData attack;
    [Tooltip("Pause in seconds before this step begins")]
    public float delayBeforeStep;
}
```

### DD-3: TelegraphVisualController on Root (Cached by EnemyAI)

`TelegraphVisualController` is a MonoBehaviour placed on the enemy root object. References the `SpriteRenderer` on the sprite child via `[SerializeField]`. `EnemyAI` caches it alongside `EnemyBase`.

**Rationale:** All enemy logic lives on root (EnemyBase, EnemyAI, BasicMeleeEnemy). Consistent placement. AttackState calls it through the `EnemyAI` context.

**Visual treatments:**
- **Normal telegraph:** Sprite tints from white → yellow over `telegraphDuration` (wind-up feel)
- **Unstoppable telegraph:** Rapid red/original blink (2–3 flashes), stays red during active frames

```csharp
public class TelegraphVisualController : MonoBehaviour
{
    [SerializeField] private SpriteRenderer _sprite;

    /// <summary>
    /// Play Normal telegraph: smooth white→yellow ramp over duration.
    /// </summary>
    public Coroutine PlayNormalTelegraph(float duration)

    /// <summary>
    /// Play Unstoppable telegraph: rapid red flashes over duration.
    /// </summary>
    public Coroutine PlayUnstoppableTelegraph(float duration)

    /// <summary>
    /// Stop any active telegraph and reset sprite color.
    /// </summary>
    public void CancelTelegraph()
}
```

### DD-4: EnemyAI Owns Pattern Selection

`EnemyAI` gets a `SelectPattern()` method that evaluates range, weight, and cooldown. `AttackState` receives the chosen pattern (or null for fallback to single attack). Selection is a decision, not execution — belongs in AI layer.

**Rationale:** Keeps AttackState focused on execution. Future states (e.g., reposition before attack) can influence pattern choice. Clean separation of concerns.

```csharp
// In EnemyAI
public EnemyAttackPattern SelectPattern()
{
    // 1. Filter attackPatterns by range condition (minRange/maxRange vs distance to target)
    // 2. Filter out patterns still on cooldown
    // 3. Weighted random from remaining candidates
    // 4. If no candidates, return null (AttackState falls back to attacks[])
}
```

### DD-5: Pattern Cooldowns via Dictionary on EnemyAI

Runtime cooldown tracking uses `Dictionary<EnemyAttackPattern, float>` mapping pattern → last-used `Time.time`. If all patterns are on cooldown, select the one with shortest remaining cooldown.

**Rationale:** Simple, no wrapper class needed. Dictionary is runtime-only state, doesn't pollute the SO.

```csharp
private readonly Dictionary<EnemyAttackPattern, float> _patternCooldowns = new();

private bool IsPatternReady(EnemyAttackPattern pattern)
{
    if (!_patternCooldowns.TryGetValue(pattern, out float lastUsed))
        return true;
    return Time.time - lastUsed >= pattern.patternCooldown;
}
```

### DD-6: Animator Trigger Mapping

Each `AttackPatternStep` fires the animator trigger matching its position in `EnemyData.attacks[]`. `AttackData` is looked up by index → fires `attack_{index+1}Trigger` (matching T024B's `attack_1Trigger`–`attack_5Trigger` convention). If the AttackData is not in `attacks[]`, defaults to `attack_1Trigger`.

**Rationale:** T024B established the trigger naming convention. Mapping by index keeps it simple and avoids adding extra fields to AttackData.

---

## File Plan

| # | File | Action | Pillar | Purpose |
|---|------|--------|--------|---------|
| 1 | `Scripts/World/EnemyAttackPattern.cs` | NEW | World | ScriptableObject: named sequence of AttackPatternStep, selection weight, range condition, cooldown |
| 2 | `Scripts/World/EnemyData.cs` | MODIFY | World | Add `EnemyAttackPattern[] attackPatterns` field alongside existing `attacks[]` |
| 3 | `Scripts/World/TelegraphVisualController.cs` | NEW | World | MonoBehaviour: Normal (yellow ramp) and Unstoppable (red flash) telegraph visuals |
| 4 | `Scripts/World/EnemyAI.cs` | MODIFY | World | Add `SelectPattern()`, `_patternCooldowns` dictionary, cache `TelegraphVisualController`, `RecordPatternUsed()` |
| 5 | `Scripts/World/States/AttackState.cs` | MODIFY | World | Execute multi-step patterns via coroutine loop; delegate visuals to TelegraphVisualController; clean early-exit on stun/death mid-pattern |
| 6 | `Editor/Prefabs/BasicEnemyPrefabCreator.cs` | MODIFY | Editor | Create 2–3 EnemyAttackPattern assets, add TelegraphVisualController, create additional hitbox children if needed |
| 7 | `Tests/EditMode/World/EnemyAttackPatternTests.cs` | NEW | Tests | Test pattern selection (weight, range, cooldown filtering), step sequencing, fallback to attacks[] |

---

## Imports (all allowed)

All World files import only from:
- `TomatoFighters.Shared.Interfaces` (IDamageable, IAttacker, IDefenseProvider)
- `TomatoFighters.Shared.Data` (AttackData, DamagePacket)
- `TomatoFighters.Shared.Enums` (TelegraphType, DamageResponse, DamageType)
- `TomatoFighters.Shared.Components` (HitboxDamage, ClashTracker)
- `System`, `System.Collections`, `System.Collections.Generic`, `UnityEngine`

No cross-pillar violations.

---

## Concrete Patterns for BasicMeleeEnemy

### Pattern 1: "Quick Jabs" (close range)
- **Steps:** BasicSlash → (0.2s delay) → BasicSlash
- **Range:** 0–1.5
- **Weight:** 2.0 (most common)
- **Cooldown:** 1.5s

### Pattern 2: "Slash" (medium range)
- **Steps:** BasicSlash (single attack)
- **Range:** 0–2.0
- **Weight:** 1.5
- **Cooldown:** 1.0s

### Pattern 3: "Heavy Slam" (close range, unstoppable)
- **Steps:** BasicHeavy (single unstoppable attack)
- **Range:** 0–1.2
- **Weight:** 0.5 (rare, punishing)
- **Cooldown:** 4.0s

---

## Risks and Gotchas

1. **AttackState coroutine complexity** — Looping through pattern steps makes the coroutine longer. Must handle clean early-exit if enemy is stunned or killed mid-pattern (check `EnemyBase.IsStunned` or death flag between steps).
2. **Hitbox ID mismatch** — Each step's AttackData references a `hitboxId`. BasicMeleeEnemy currently has one hitbox child (`Hitbox_Punch`). If patterns use AttackData with different hitboxIds, the Creator Script must create matching hitbox children.
3. **Animator trigger mapping** — AttackData doesn't carry an animator trigger index. Mapping by position in `EnemyData.attacks[]` works but is fragile. If attack order changes, triggers shift. Document this clearly.
4. **TestDummyEnemy unchanged** — Has its own attack loop. Won't benefit from patterns or TelegraphVisualController, but won't break either (Option B fallback).

---

## Acceptance Criteria

- [ ] `EnemyAttackPattern` ScriptableObject with flat step sequence, weight, range, cooldown
- [ ] `TelegraphVisualController` with distinct Normal (yellow ramp) vs Unstoppable (red flash) visuals
- [ ] `AttackState` executes multi-step patterns with per-step delay and telegraph
- [ ] Clean early-exit from pattern on stun/death
- [ ] `EnemyAI.SelectPattern()` with range filtering, weight-based selection, cooldown tracking
- [ ] Fallback to `attacks[]` when no patterns defined
- [ ] BasicMeleeEnemy has 2–3 concrete patterns via Creator Script
- [ ] Edit mode tests for pattern selection logic
- [ ] Pillar boundaries clean (World + Shared only)
- [ ] No changes to TestDummyEnemy

---

## Execution Order

1. `EnemyAttackPattern.cs` — new SO, no dependencies
2. `EnemyData.cs` — add `attackPatterns[]` field
3. `TelegraphVisualController.cs` — new component, no dependencies
4. `EnemyAI.cs` — add `SelectPattern()`, cache telegraph controller, cooldown dictionary
5. `AttackState.cs` — modify to execute multi-step patterns, delegate telegraph visuals
6. `BasicEnemyPrefabCreator.cs` — create pattern assets, wire telegraph, extra hitboxes
7. `EnemyAttackPatternTests.cs` — test selection + sequencing
