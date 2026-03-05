# T032: BossAI Framework

> **Phase:** 3 | **Priority:** P0 | **Owner:** Dev 3 (World) | **Depends on:** T022, T026
> **Blocks:** T042
> **Status:** DONE | **Branch:** `pillar3/T032-boss-ai-framework` | **Completed:** 2026-03-05

---

## Summary

Phase-based boss AI system that extends the existing EnemyAI state machine. Bosses transition through HP%-threshold phases, each with its own attack pattern list (SO-driven), tempo multiplier, and behavior flags. Big attacks leave configurable punish windows. Camera zoom on stun is already wired via T026. Phase transitions fire a dedicated SO event channel for camera/UI integration. Deliverable includes one test boss with 3 phases and a Creator Script for the prefab.

---

## Design Decisions

### DD-1: BossAI as Companion Component (not subclass)

BossAI is a separate `[RequireComponent(typeof(EnemyAI))]` MonoBehaviour that sits alongside EnemyAI. It monitors HP%, swaps attack pools, and injects phase-specific states. EnemyAI is not subclassed.

**Rationale:** EnemyAI already owns the state machine and targeting. BossAI adds a layer on top — regular enemies keep working unchanged, BossAI is opt-in. Mini-bosses (T042) can use a simplified version of the same component.

```csharp
[RequireComponent(typeof(EnemyAI))]
public class BossAI : MonoBehaviour
{
    [SerializeField] private BossData bossData;
    private EnemyAI enemyAI;
    private EnemyBase enemyBase;
    private int currentPhaseIndex = 0;

    private void OnDamaged()
    {
        float hpPercent = enemyBase.CurrentHealth / enemyBase.MaxHealth;
        CheckPhaseTransition(hpPercent);
    }
}
```

### DD-2: BossPhaseData as [Serializable] Inline on BossData SO

All phases for a boss are authored on a single BossData SO as an inline `BossPhaseData[]` array. Each phase has its own attack pool, tempo multiplier, and HP threshold. No separate SO per phase.

**Rationale:** One asset per boss, all phases visible together. Phases don't need to be shared across bosses. Simpler to author and inspect.

```csharp
[System.Serializable]
public class BossPhaseData
{
    public string phaseName;
    [Range(0f, 1f)] public float hpThreshold;       // HP% at which this phase activates
    public AttackData[] attacks;                      // This phase's attack pool
    [Range(0.5f, 2f)] public float tempoMultiplier;  // Multiplier on attack cooldown speed
    public float attackCooldownOverride;              // -1 = use base EnemyData value
    public bool enableEnrage;                         // Visual indicator (sprite tint)
}
```

### DD-3: Punish Windows via AttackData Flag (Option A)

Punish windows are driven by per-attack flags on AttackData. AttackState checks `attack.hasPunishWindow` after its coroutine finishes and transitions to BossPunishState instead of ChaseState.

**Rationale:** Data-driven (SO philosophy), per-attack granularity, works for non-boss enemies and mini-bosses too. AttackState already reads from AttackData for everything else.

```csharp
// Added to AttackData.cs
[Header("Punish Window")]
public bool hasPunishWindow;
[Tooltip("How long the attacker is vulnerable after this attack")]
public float punishWindowDuration = 1.0f;
```

### DD-4: Phase Transition — Skip to Correct Phase

When a massive hit crosses multiple HP thresholds at once, the boss skips straight to the correct phase (one transition cinematic). No cascading through intermediate phases.

**Rationale:** Cascading gives unearned invulnerability time. The player earned that burst damage. Single transition is cleaner and faster.

Phase transition state: ~1.5s invulnerable pause with visual flash (sprite blink white). Duration configurable on BossData.

### DD-5: Minimal EnemyAI Modifications (Backwards-Compatible)

EnemyAI gets three additions:
- `attackPoolOverride` (nullable) — `GetAvailableAttacks()` returns this when set, otherwise `Data.attacks`
- `tempoMultiplier` (default 1.0) — AttackState applies to cooldown
- Public setters: `SetAttackPool(AttackData[])`, `SetTempoMultiplier(float)`

AttackState changes:
- Reads from `AI.GetAvailableAttacks()` instead of `AI.Data.attacks`
- Checks `attack.hasPunishWindow` after coroutine — transitions to BossPunishState if true
- Applies `AI.TempoMultiplier` to attack cooldown

All backwards-compatible — regular enemies unchanged (override is null, tempo is 1.0).

### DD-6: OnBossPhaseChanged SO Event Channel

A dedicated `VoidEventChannel` (or typed event with phase index) fires on phase transitions. CameraController2D can listen for a dramatic zoom. HUD can show phase indicator. Decoupled from BossAI internals.

```csharp
// On BossAI
[SerializeField] private VoidEventChannel onBossPhaseChanged;
```

---

## File Plan

| # | File | Pillar | Action | Purpose |
|---|------|--------|--------|---------|
| 1 | `Scripts/World/BossPhaseData.cs` | World | **New** | `[Serializable]` data class — HP% threshold, attack list, tempo multiplier, enrage flag |
| 2 | `Scripts/World/BossData.cs` | World | **New** | ScriptableObject — extends `EnemyData` conceptually (or standalone with `EnemyData` reference), holds `BossPhaseData[]`, phase transition duration, punish config |
| 3 | `Scripts/Shared/Data/AttackData.cs` | Shared | **Modified** | Add `hasPunishWindow` bool + `punishWindowDuration` float |
| 4 | `Scripts/World/EnemyAI.cs` | World | **Modified** | Add `GetAvailableAttacks()`, `SetAttackPool()`, `SetTempoMultiplier()`, `tempoMultiplier` field |
| 5 | `Scripts/World/States/AttackState.cs` | World | **Modified** | Read from `AI.GetAvailableAttacks()`, apply tempo multiplier, check punish window → BossPunishState |
| 6 | `Scripts/World/States/BossPunishState.cs` | World | **New** | Post-big-attack vulnerability state, configurable duration, sets `IsInPunishableState` |
| 7 | `Scripts/World/States/BossPhaseTransitionState.cs` | World | **New** | Invulnerable cinematic pause (~1.5s), visual flash, fires phase change event |
| 8 | `Scripts/World/BossAI.cs` | World | **New** | Phase monitor — HP% checks, attack pool swapping, phase transition injection, SO event firing |
| 9 | `Scripts/World/BossEnemy.cs` | World | **New** | Concrete boss class extending EnemyBase, bridges EnemyBase + EnemyAI + BossAI |
| 10 | `Editor/Prefabs/BossPrefabCreator.cs` | Editor | **New** | Creator Script — test boss prefab + BossData SO + 6 boss AttackData SOs |

---

## Test Boss Design

| Phase | HP Threshold | Tempo | Attacks | Notes |
|-------|-------------|-------|---------|-------|
| **Phase 1** | 100%–60% | 1.0x | BossSlash, BossOverhead | Slow, predictable, teaches patterns |
| **Phase 2** | 60%–30% | 1.3x | + BossLunge, BossUnstoppableSlam | Adds charging lunge + one unstoppable attack |
| **Phase 3** | 30%–0% | 1.6x | + BossGroundPound (punish window) | Enraged (red tint), ground pound leaves 1.5s punish window |

**Test Boss Stats:**
- HP: 300 (vs 80 for BasicMeleeEnemy)
- Pressure Threshold: 80 (vs 40)
- Knockback Resistance: 0.5 (vs 0.2)
- Movement Speed: 4.5 (slightly slower than basic enemy's 5.5)
- Phase Transition Duration: 1.5s (invulnerable)

**AttackData SOs (6 total):**

| Attack | Damage | Knockback | Telegraph | Punish | Type |
|--------|--------|-----------|-----------|--------|------|
| BossSlash | 0.8x | (2.5, 0) | Normal | No | Normal |
| BossOverhead | 1.2x | (3, 0.5) | Normal | No | Normal |
| BossLunge | 1.0x | (4, 0) | Normal | No | Normal |
| BossUnstoppableSlam | 1.5x | (3, 1) | Unstoppable | No | Unstoppable |
| BossGroundPound | 2.0x | (5, 1.5) | Unstoppable | Yes (1.5s) | Unstoppable |
| BossEnragedSlash | 1.0x | (3, 0) | Normal | No | Normal (faster) |

**Visual:** Larger dark-red debug square (1.2x scale of BasicMeleeEnemy), red tint in Phase 3.

---

## Imports (all allowed)

All World files import only from:
- `TomatoFighters.Shared.Interfaces` (IDamageable, IAttacker)
- `TomatoFighters.Shared.Data` (AttackData, DamagePacket)
- `TomatoFighters.Shared.Enums` (TelegraphType, DamageType)
- `TomatoFighters.Shared.Events` (VoidEventChannel)
- `System`, `System.Collections`, `UnityEngine`

No cross-pillar violations.

---

## Acceptance Criteria

- [ ] Phase system with HP% transitions (3 phases on test boss)
- [ ] Per-phase attack patterns (BossPhaseData on BossData SO)
- [ ] Punish windows after big attacks (configurable via AttackData)
- [ ] Camera zoom on stun (existing T026 wiring — verified working for boss)
- [ ] OnBossPhaseChanged SO event for camera/UI integration
- [ ] Phase transition cinematic (invulnerable pause + visual flash)
- [ ] Multi-threshold skip (no cascading)
- [ ] One test boss with 3 phases via Creator Script
- [ ] Backwards-compatible EnemyAI changes (regular enemies unaffected)
- [ ] Pillar boundaries clean (World + Shared only)

---

## Execution Order

1. `BossPhaseData.cs` — plain data class, no dependencies
2. `BossData.cs` — SO holding phases, depends on BossPhaseData + AttackData
3. **Modify** `AttackData.cs` — add `hasPunishWindow` + `punishWindowDuration`
4. **Modify** `EnemyAI.cs` — add `GetAvailableAttacks()`, `SetAttackPool()`, `SetTempoMultiplier()`
5. **Modify** `AttackState.cs` — read from `GetAvailableAttacks()`, apply tempo, check punish
6. `BossPunishState.cs` — post-big-attack vulnerability state
7. `BossPhaseTransitionState.cs` — cinematic pause state
8. `BossAI.cs` — phase monitor, HP% checks, attack pool swapping
9. `BossEnemy.cs` — concrete boss class
10. `BossPrefabCreator.cs` — Creator Script for prefab + all SOs
