# T015: HitboxManager

**Phase:** 2 | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T014
**Status:** DONE
**Branch:** `pillar1/T015-hitbox-manager`
**Completed:** 2026-03-03

---

## Summary

Activate/deactivate attack colliders via Animation Events. Never in Update(). Reports hit-confirm to ComboSystem for cancel enabling. Supports multiple hitbox shapes per attack via named child GameObjects on the player prefab. Collision layer filtering separates player attacks from enemy attacks. Unlocks cancel system end-to-end testing — wires collision → `ComboController.OnHitConfirmed()` → cancel window opens.

HitboxDamage is **detection-only**. It fires events when collisions occur. A temporary damage shim in HitboxManager applies damage directly until T016 (DefenseSystem) takes over resolution.

## Acceptance Criteria

- [ ] Animation Event-driven activation (never Update)
- [ ] Hit-confirm callback to ComboSystem via event chain
- [ ] Multiple hitbox shapes supported (child GOs with Collider2D)
- [ ] Layer filtering (PlayerHitbox ↔ EnemyHurtbox, EnemyHitbox ↔ PlayerHurtbox)
- [ ] HashSet per-activation hit tracking (no double-hits, allows multi-target)
- [ ] Loud error when AttackData references a hitboxId that doesn't exist
- [ ] Temporary damage shim clearly marked for T016 replacement
- [ ] Cancel system integration tests: hit → cancel window → dash/jump cancel executes

## Design Decisions

### DD-1: Child GameObjects with named Collider2Ds

Each hitbox shape is a child GameObject on the player prefab with a Collider2D (Box, Circle, Polygon) set to trigger mode, disabled by default. Designers can visually place/resize hitboxes in the Scene view. ~3-5 shapes per character cover all combo steps.

```
Player (root)
├── HitboxManager
├── Hitbox_Jab          ← small BoxCollider2D, slightly in front
├── Hitbox_Sweep        ← wide BoxCollider2D, low and wide
├── Hitbox_Uppercut     ← tall BoxCollider2D, angled upward
├── Hitbox_Slam         ← large CircleCollider2D, below character
└── Hitbox_Lunge        ← narrow BoxCollider2D, extended forward
```

**Rationale:** More designer-friendly than programmatic collider resizing. Supports arbitrary shapes (circle, polygon, capsule). Multiple attacks can reuse the same hitbox shape by referencing the same `hitboxId`.

### DD-2: AttackData holds hitboxId, Animation Events are generic

AttackData SO gets a new `string hitboxId` field (e.g. `"Jab"`, `"Sweep"`). Animation Events call a generic `ActivateHitbox()` with no string argument. HitboxManager reads `hitboxId` from the current ComboStep's AttackData to know which child to enable.

```csharp
// Animation Event calls this — no arguments
public void ActivateHitbox()
{
    var attackData = GetCurrentAttackData();
    var hitbox = FindHitbox(attackData.hitboxId);
    if (hitbox == null)
    {
        Debug.LogError($"[HitboxManager] No hitbox found with id '{attackData.hitboxId}' " +
            $"on {gameObject.name}. Check AttackData '{attackData.attackName}' or prefab children.", this);
        return;
    }
    hitbox.gameObject.SetActive(true);
}
```

**Rationale:** Keeps data in SOs (project convention). Hitbox mapping can be bulk-edited without touching animation clips. Animation clips stay generic and reusable.

### DD-3: HashSet hit tracking on HitboxDamage

Each hitbox child GO has a `HitboxDamage` MonoBehaviour. It tracks hits per activation using `HashSet<IDamageable>`, cleared on `OnEnable` (when the hitbox activates for a new swing). Uses `OnTriggerEnter2D` for detection. Tracks by `IDamageable` (not GameObject) to prevent double-damage from entities with multiple colliders.

```csharp
public class HitboxDamage : MonoBehaviour
{
    private HashSet<IDamageable> _hitThisActivation = new();

    private void OnEnable()
    {
        _hitThisActivation.Clear();
    }

    private void OnTriggerEnter2D(Collider2D other)
    {
        var target = other.GetComponent<IDamageable>();
        if (target == null) return;
        if (!_hitThisActivation.Add(target)) return;

        OnHitDetected?.Invoke(new HitDetectionData { ... });
    }
}
```

**Rationale:** HashSet allows hitting multiple different enemies per swing (beat 'em up requirement) while preventing the same enemy being hit twice per activation. Clearing on OnEnable (not OnDisable) ensures a fresh set at the start of each swing.

### DD-4: Event-driven hit-confirm chain

HitboxDamage fires `OnHitDetected` event. HitboxManager subscribes and forwards to `ComboController.OnHitConfirmed()`. Clean separation — HitboxDamage doesn't know about combos, and other systems (hitstop, VFX) can subscribe later without modifying HitboxDamage.

```
HitboxDamage.OnHitDetected
        ↓
HitboxManager (subscribed)
        ↓
ComboController.OnHitConfirmed()
        ↓
Cancel window opens → buffered dash/jump executes
```

### DD-5: Detection-only pattern with temporary damage shim

HitboxDamage is **pure detection**. It reports "this attack collider touched this target" and nothing else. It does NOT resolve damage, check defense state, or build DamagePackets.

The **target decides the outcome** of a collision (Hit, Deflected, Clashed, Dodged). This resolution is T016's responsibility (DefenseSystem).

Until T016 exists, HitboxManager contains a temporary shim that applies damage directly:

```csharp
// TEMPORARY: Direct damage until T016 (DefenseSystem) adds resolution
private void HandleHitDetected(HitDetectionData data)
{
    if (data.target.IsInvulnerable) return;

    var packet = BuildDamagePacket(data.attackData);
    data.target.TakeDamage(packet);
    comboController.OnHitConfirmed();
}
```

T016 will replace this shim with proper deflect/clash/dodge resolution. The shim is clearly marked with comments for easy identification.

### DD-6: Four physics layers

| Layer | Purpose |
|-------|---------|
| `PlayerHitbox` | Attack colliders on player characters |
| `EnemyHurtbox` | Hurtboxes on enemies (can-be-hit zones) |
| `EnemyHitbox` | Attack colliders on enemies (T016 — deflect detection) |
| `PlayerHurtbox` | Hurtbox on player (for receiving enemy attacks) |

Collision matrix:
- `PlayerHitbox` ↔ `EnemyHurtbox` = **yes**
- `EnemyHitbox` ↔ `PlayerHurtbox` = **yes**
- All other hitbox/hurtbox combinations = **no**
- All hitbox/hurtbox layers ↔ `Default` = **no**

**Rationale:** If `OnTriggerEnter2D` fires at all, it's guaranteed to be a valid combat target. No tag checks or GetComponent guessing needed beyond IDamageable.

## File Plan

| File | Action | Description |
|------|--------|-------------|
| `Scripts/Combat/Hitbox/HitDetectionData.cs` | CREATE | Struct: `IDamageable target`, `AttackData attackData`, `Vector2 hitPoint`, `CharacterType attacker` |
| `Scripts/Combat/Hitbox/HitboxDamage.cs` | CREATE | MonoBehaviour on each hitbox child. HashSet tracking, OnTriggerEnter2D, fires `OnHitDetected` event |
| `Scripts/Combat/Hitbox/HitboxManager.cs` | CREATE | MonoBehaviour on player root. Orchestrates activation/deactivation via Animation Events. Subscribes to HitboxDamage events, forwards hit-confirm to ComboController. Contains temporary damage shim |
| `Scripts/Shared/Data/AttackData.cs` | MODIFY | Add `string hitboxId` field |
| `Prefabs/Player/Player.prefab` | MODIFY | Add HitboxManager component, add hitbox child GOs with Collider2D + HitboxDamage |
| Attack animation clips | MODIFY | Add Animation Events calling `ActivateHitbox()` / `DeactivateHitbox()` |

## Execution Order

1. Add `hitboxId` field to `AttackData.cs`
2. Create `HitDetectionData.cs` struct
3. Create `HitboxDamage.cs` (self-contained, testable independently)
4. Create `HitboxManager.cs` (wires HitboxDamage → ComboController, contains temp damage shim)
5. Set up 4 physics layers in Unity project settings (PlayerHitbox, EnemyHurtbox, EnemyHitbox, PlayerHurtbox)
6. Configure collision matrix
7. Update `Player.prefab` with hitbox child GOs + components
8. Add Animation Events to attack clips
9. End-to-end test: attack → hitbox activates → hit detected → hit-confirm → cancel window opens

## Dependencies

- **Depends on:** T014 (ComboSystem — all 4 characters' AttackData + ComboDefinitions)
- **Blocks:** T016 (DefenseSystem — needs hit detection to resolve deflect/clash/dodge), T017 (Character Passives — Bloodlust needs hit-confirm)
