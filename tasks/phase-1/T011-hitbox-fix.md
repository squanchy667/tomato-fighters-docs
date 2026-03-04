# T011-FIX: Bidirectional Damage — Hitbox Activation Without Animation Events

## Metadata
| Field | Value |
|-------|-------|
| **Phase** | 1 — Foundation |
| **Type** | bugfix + refactor |
| **Priority** | P0 |
| **Owner** | Dev 1 |
| **Agent** | combat-agent |
| **Depends On** | T011 (EnemyBase — completed) |
| **Blocks** | T013 (Test Scene), T016 (DefenseSystem) |
| **Status** | SPEC READY |
| **Branch** | `pillar3/T011-enemy-base` (continue on existing branch) |

## Problem Statement

Player attack hitboxes never activate when pressing the attack button. Three breaks in the chain:

1. **No Animation Events on attack clips** — `HitboxManager.ActivateHitbox()` is never called
2. **`ComboStep.attackData` is null** — `PlayerPrefabCreator` creates steps without AttackData SO references
3. **HitboxManager not on Player prefab** — Only added via manual `SetupMysticaCharacter` script

## Objective

Fix bidirectional damage (player ↔ enemy) without requiring animation event setup. Refactor PlayerPrefabCreator into a generic system so each character type has its own creator script (Mystica, Brutor, Slasher, Viper) without code duplication.

End state: A test scene where the enemy's timer-based attacks reduce the player's HP, and the player's combo attacks reduce the enemy's HP.

## Design Decisions

### DD-1: Timer fallback inside HitboxManager (not a separate component)
**Rationale:** Adding `useTimerFallback` directly to HitboxManager reuses the same `ActivateHitbox()`/`DeactivateHitbox()` methods. ~15 lines, clearly marked as temporary, easy to disable when animation events are ready. A separate bridge component would add file bloat for temporary functionality.

```csharp
[Header("Timer Fallback (no Animation Events)")]
[SerializeField] private bool useTimerFallback;
[SerializeField] private float fallbackActiveDuration = 0.3f;
```

### DD-2: Fixed 0.3s fallback duration
**Rationale:** Simple, predictable, easy to tune in Inspector. Per-attack frame data in AttackData SOs may not be accurate yet. Good enough for testing without animations.

### DD-3: Generic PlayerPrefabCreator + character-specific creators
**Rationale:** PlayerPrefabCreator currently has Mystica-specific code (movement config values, combo steps, SO paths). Extracting character-specific logic into MysticaCharacterCreator allows future BrutorCharacterCreator, SlasherCharacterCreator, ViperCharacterCreator without code duplication.

Architecture:
```
PlayerPrefabCreator (generic utility)
├── CreatePlayerPrefab(CharacterPrefabConfig) — entry point
├── SetupPrefab() — structure, components, hitboxes, HitboxManager
└── Helpers (EnsureComponent, FindOrCreateChild, etc.)

CharacterPrefabConfig (data class)
├── characterType, prefabPath
├── movementConfig, comboDefinition, animatorController
├── hitboxDefinitions[] — shapes, sizes, offsets, hitboxIds
├── baseAttack, useTimerFallback, fallbackActiveDuration

MysticaCharacterCreator (Mystica-specific)
├── [MenuItem("TomatoFighters/Characters/Create Mystica")]
├── Creates/loads Mystica SOs (movement config, combo def, AttackData)
├── Wires AttackData into ComboSteps
├── Defines Mystica hitbox shapes (Burst, BigBurst, Bolt)
└── Calls PlayerPrefabCreator.CreatePlayerPrefab(config)
```

### DD-4: SetupMysticaCharacter.cs absorbed into MysticaCharacterCreator
**Rationale:** SetupMysticaCharacter was a manual workaround that only added hitboxes and HitboxManager. MysticaCharacterCreator does everything in one step. Delete SetupMysticaCharacter to avoid confusion.

### DD-5: Layer collision matrix set in MovementTestSceneCreator
**Rationale:** The editor script can call `Physics2D.IgnoreLayerCollision()` to ensure the correct layer pairs collide. This modifies ProjectSettings globally but is idempotent and safe to re-run.

## File Plan

| # | File | Action | Location | Purpose |
|---|------|--------|----------|---------|
| 1 | `HitboxManager.cs` | **Modify** | `Scripts/Combat/Hitbox/` | Add timer fallback mode (~20 lines) |
| 2 | `CharacterPrefabConfig.cs` | **New** | `Editor/Prefabs/` | Config data class: character type, hitbox defs, attack params |
| 3 | `PlayerPrefabCreator.cs` | **Refactor** | `Editor/Prefabs/` | Make generic — accept config, create hitbox children, add HitboxManager |
| 4 | `MysticaCharacterCreator.cs` | **New** | `Editor/Characters/` | Mystica-specific SO creation + config, delegates to PlayerPrefabCreator |
| 5 | `MovementTestSceneCreator.cs` | **Modify** | `Editor/Prefabs/` | Add layer collision matrix setup for hitbox/hurtbox pairs |
| 6 | `SetupMysticaCharacter.cs` | **Delete** | `Editor/` | Functionality absorbed into MysticaCharacterCreator |

## Execution Order

1. **HitboxManager.cs** — Add timer fallback (no dependencies on other changes)
2. **CharacterPrefabConfig.cs** — Create config data class with HitboxDefinition struct
3. **PlayerPrefabCreator.cs** — Refactor to generic:
   - Accept `CharacterPrefabConfig` parameter
   - Move Mystica-specific SO creation out
   - Add hitbox child creation from `HitboxDefinition[]`
   - Add HitboxManager component wiring
   - Add debug visual for hitbox children (semi-transparent colored sprite)
4. **MysticaCharacterCreator.cs** — Mystica-specific:
   - `CreateOrLoadMysticaMovementConfig()` (moved from PlayerPrefabCreator)
   - `CreateOrLoadMysticaComboDefinition()` (moved, now wires AttackData)
   - Load existing Mystica AttackData SOs and assign to ComboSteps
   - Define Mystica hitbox shapes
   - Call `PlayerPrefabCreator.CreatePlayerPrefab(config)`
5. **MovementTestSceneCreator.cs** — Add `SetupLayerCollisionMatrix()`:
   - Enable PlayerHitbox ↔ EnemyHurtbox collision
   - Enable EnemyHitbox ↔ PlayerHurtbox collision
   - Disable self-collisions for all hitbox/hurtbox layers
6. **Delete SetupMysticaCharacter.cs** (and its .meta)
7. **Verify in Unity:**
   - Menu: TomatoFighters > Characters > Create Mystica
   - Menu: TomatoFighters > Create Test Scene
   - Play → enemy attacks reduce player HP, player attacks reduce enemy HP

## Requirements Detail

### 1. HitboxManager Timer Fallback

Add to `HitboxManager.cs`:
```csharp
[Header("Timer Fallback (no Animation Events)")]
[Tooltip("Auto-activate hitboxes on combo step start. Disable when animation events are set up.")]
[SerializeField] private bool useTimerFallback;

[Tooltip("How long hitbox stays active per attack in fallback mode.")]
[SerializeField] private float fallbackActiveDuration = 0.3f;

private Coroutine _fallbackCoroutine;
```

- In `OnEnable()`: if `useTimerFallback`, subscribe to `comboController.AttackStarted`
- In `OnDisable()`: unsubscribe from `comboController.AttackStarted`
- Handler `OnAttackStartedFallback(int stepIndex)`:
  - Stop existing fallback coroutine if running
  - Call `ActivateHitbox()`
  - Start coroutine: wait `fallbackActiveDuration` → call `DeactivateActiveHitbox()`

### 2. CharacterPrefabConfig

```csharp
public enum HitboxShape { Circle, Box }

[Serializable]
public struct HitboxDefinition
{
    public string hitboxId;
    public HitboxShape shape;
    public float circleRadius;
    public Vector2 boxSize;
    public Vector2 offset;
}

public class CharacterPrefabConfig
{
    public string prefabPath;
    public CharacterType characterType;
    public MovementConfig movementConfig;
    public ComboDefinition comboDefinition;
    public AnimatorController animatorController;
    public InputActionAsset inputActions;
    public HitboxDefinition[] hitboxes;
    public float baseAttack = 10f;
    public bool useTimerFallback = true;
    public float fallbackActiveDuration = 0.3f;
}
```

### 3. PlayerPrefabCreator (Generic)

Refactor `SetupPrefab()` to accept `CharacterPrefabConfig`:
- All existing component setup remains (Rigidbody2D, colliders, sprite, motor, combo, input, etc.)
- **NEW:** Create hitbox children from `config.hitboxes[]`:
  - Name: `Hitbox_{hitboxId}`
  - Layer: PlayerHitbox
  - Collider: Circle or Box per definition
  - Component: HitboxDamage (from Shared)
  - Debug visual: Semi-transparent red sprite matching collider area
  - Start disabled
- **NEW:** Add HitboxManager:
  - Wire `comboController` reference
  - Set `baseAttack` from config
  - Set `useTimerFallback` and `fallbackActiveDuration` from config
- Remove Mystica-specific `CreateOrLoadMysticaMovementConfig()` and `CreateOrLoadMysticaComboDefinition()`
- Keep generic helpers: `EnsureComponent`, `FindOrCreateChild`, `WireInputAction`, `RepairHitboxChildren`, `RemoveMissingScripts`, `EnsureFolderExists`

### 4. MysticaCharacterCreator

Menu item: `TomatoFighters/Characters/Create Mystica`

Responsibilities:
- `CreateOrLoadMysticaMovementConfig()` — same values as current PlayerPrefabCreator
- `CreateOrLoadMysticaComboDefinition()` — same steps, BUT also loads and assigns AttackData:
  - Step 0 (Light1) → MysticaStrike1 (hitboxId: "Burst")
  - Step 1 (Light2) → MysticaStrike2 (hitboxId: "Burst")
  - Step 2 (LightFinisher) → MysticaStrike3 (hitboxId: "BigBurst")
  - Step 3 (Heavy1) → MysticaArcaneBolt (hitboxId: "Bolt")
  - Step 4 (HeavyFinisher) → MysticaEmpoweredBolt (hitboxId: "Bolt")
- Mystica hitbox definitions:
  - `Hitbox_Burst`: Circle r=0.5, offset (0.5, 0.1)
  - `Hitbox_BigBurst`: Circle r=0.8, offset (0.4, 0.1)
  - `Hitbox_Bolt`: Box 1.6x0.35, offset (1.1, 0.15)
- Build `CharacterPrefabConfig` and call `PlayerPrefabCreator.CreatePlayerPrefab(config)`

### 5. MovementTestSceneCreator Layer Setup

Add `SetupLayerCollisionMatrix()` called at the start of `CreateTestScene()`:
```csharp
private static void SetupLayerCollisionMatrix()
{
    int playerHitbox = LayerMask.NameToLayer("PlayerHitbox");
    int playerHurtbox = LayerMask.NameToLayer("PlayerHurtbox");
    int enemyHitbox = LayerMask.NameToLayer("EnemyHitbox");
    int enemyHurtbox = LayerMask.NameToLayer("EnemyHurtbox");

    if (playerHitbox < 0 || playerHurtbox < 0 || enemyHitbox < 0 || enemyHurtbox < 0)
    {
        Debug.LogError("Missing physics layers. Add PlayerHitbox, PlayerHurtbox, EnemyHitbox, EnemyHurtbox.");
        return;
    }

    // Enable cross-team collisions
    Physics2D.IgnoreLayerCollision(playerHitbox, enemyHurtbox, false);
    Physics2D.IgnoreLayerCollision(enemyHitbox, playerHurtbox, false);

    // Disable same-team and self collisions
    Physics2D.IgnoreLayerCollision(playerHitbox, playerHurtbox, true);
    Physics2D.IgnoreLayerCollision(enemyHitbox, enemyHurtbox, true);
    Physics2D.IgnoreLayerCollision(playerHitbox, playerHitbox, true);
    Physics2D.IgnoreLayerCollision(enemyHitbox, enemyHitbox, true);
    Physics2D.IgnoreLayerCollision(playerHurtbox, playerHurtbox, true);
    Physics2D.IgnoreLayerCollision(enemyHurtbox, enemyHurtbox, true);
}
```

## Acceptance Criteria

- [ ] Player attacks activate hitbox colliders via timer fallback (0.3s per attack)
- [ ] HitboxManager.useTimerFallback toggle works — unchecking disables timer-based activation
- [ ] Player hitbox collides with enemy hurtbox → enemy takes damage, HP decreases
- [ ] Enemy hitbox collides with player hurtbox → player takes damage, HP decreases
- [ ] DebugHealthBar shows HP changes for both player and enemy
- [ ] PlayerPrefabCreator is fully generic — no Mystica-specific code
- [ ] MysticaCharacterCreator produces a complete Mystica prefab in one step
- [ ] CharacterPrefabConfig supports Circle and Box hitbox shapes
- [ ] AttackData SOs wired into all ComboSteps
- [ ] Layer collision matrix correctly configured (PlayerHitbox↔EnemyHurtbox, EnemyHitbox↔PlayerHurtbox)
- [ ] SetupMysticaCharacter.cs deleted (functionality in MysticaCharacterCreator)
- [ ] No cross-pillar import violations
- [ ] Compiles with zero errors

## Layer Collision Matrix (Required)

| | PlayerHitbox | PlayerHurtbox | EnemyHitbox | EnemyHurtbox |
|---|---|---|---|---|
| **PlayerHitbox** | no | no | — | **YES** |
| **PlayerHurtbox** | no | no | **YES** | — |
| **EnemyHitbox** | — | **YES** | no | no |
| **EnemyHurtbox** | **YES** | — | no | no |

## Test Verification Steps

1. Open Unity
2. Menu: TomatoFighters > Characters > Create Mystica
3. Menu: TomatoFighters > Create Test Scene
4. Enter Play Mode
5. **Enemy → Player:** Wait for enemy timer attack (3s). See enemy telegraph (white flash), then attack (red-orange). Player HP bar should decrease. Console log: "Player took X damage."
6. **Player → Enemy:** Press attack (LMB). See hitbox activate briefly. If close enough, enemy HP bar should decrease. Console log: "TestDummy took X damage."
7. Both DebugHealthBars update in real-time.

## References

- [T011 Spec](./T011-enemy-base.md) — Original EnemyBase implementation
- [T015 Spec](./T015-hitbox-manager.md) — HitboxManager system (if exists)
- [System Overview](../../architecture/system-overview.md)
- [Data Flow](../../architecture/data-flow.md)
