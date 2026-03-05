# T025: HUD — Health, Mana, Combo Counter

- **Phase:** 2 | **Priority:** P1 | **Owner:** Dev 3 (World)
- **Depends On:** T007 (CharacterStatCalculator), T011 (EnemyBase), T026 (PressureSystem)
- **Blocks:** T013 (visual feedback for test scene), T032 (boss HP phases), T045 (feel pass)

---

## Summary

Screen-space overlay HUD for player stats + world-space enemy health bars. Establishes the UI pillar for all future UI work (T019, T031, T040).

## Deliverables

1. **Player health bar** — % of max HP, animates on damage
2. **Player mana bar** — current/max mana with regen visualization
3. **Combo counter** — confirmed hit count with decay timer
4. **Path indicator** — Main/Secondary path icons (empty slots when unselected)
5. **Enemy world-space health bars** — floating above enemies, includes pressure meter
6. **PlayerManaTracker** — runtime mana tracking in Shared (both Combat and World need it)
7. **FloatEventChannel** — new SO event channel type for float payloads
8. **HUDCreator** — Editor script to build HUD Canvas + enemy health bar prefabs

---

## Design Decisions

### DD-1: Health/Mana Data Flow — SO Event Channels

HUD (World pillar) needs data from PlayerDamageable (Combat pillar). Direct reference violates pillar boundaries.

**Decision:** Use SO event channels. `PlayerDamageable` fires `FloatEventChannel` assets (`OnPlayerHealthChanged`, `OnPlayerManaChanged`) whenever values change. HUD subscribes to the same SO assets. Fully decoupled — other systems (camera shake on low HP, etc.) can also subscribe.

**Rationale:** Matches existing architecture (WaveManager already uses `VoidEventChannel` and `IntEventChannel`). Avoids Update() polling.

### DD-2: Combo Counter Source — New IntEventChannel

**Decision:** Add a new `IntEventChannel` SO asset (`OnComboHitConfirmed`). `ComboController` raises it on hit-confirm with the current combo length. HUD subscribes via the SO channel.

**Rationale:** Keeps the World pillar decoupled from Combat's `ICombatEvents` interface. The combo counter shows confirmed hits (not attacks thrown), so it fires after `HitboxManager` reports a hit to `ComboController`.

### DD-3: Enemy Health Bar Spawning — Self-Contained on EnemyBase

**Decision:** Each `EnemyBase` spawns its own world-space health bar in `Awake()` by instantiating a prefab referenced via `[SerializeField]`. No HUDManager involvement.

**Rationale:** Self-contained, works for any enemy regardless of spawn source (WaveManager, scripted, editor-placed). No coupling to WaveManager events.

### DD-4: PlayerManaTracker — Create Now in Shared

**Decision:** Create `Scripts/Shared/Components/PlayerManaTracker.cs` (~40-50 lines). Tracks `currentMana`, handles per-second regen, fires `FloatEventChannel` on change. Reads max mana from `CharacterBaseStats` (or `FinalStats` when wired).

**Rationale:** Combat pillar needs mana tracking for T028 (Path Abilities) anyway. Building it now makes the mana bar functional immediately. Placed in Shared because both Combat (consume mana) and World (display mana) interact with it.

```csharp
public class PlayerManaTracker : MonoBehaviour
{
    [SerializeField] private CharacterBaseStats baseStats;
    [SerializeField] private FloatEventChannel onManaChanged; // fires normalized 0-1
    [SerializeField] private FloatEventChannel onManaRegenTick; // optional, for regen VFX

    public float CurrentMana { get; private set; }
    public float MaxMana => baseStats.mana;

    public bool TryConsume(float amount) { ... }
    public void Restore(float amount) { ... }
}
```

### DD-5: Pressure Meter on Enemy Health Bars — Include

**Decision:** Enemy world-space bars show both health (green→red) and pressure (blue fill below health bar). When pressure threshold is reached and enemy is stunned, the pressure bar flashes.

**Rationale:** T026 (PressureSystem) is already DONE. It's a second `Image.fillAmount` on the same prefab — minimal extra code, big payoff for combat testing.

---

## File Plan

### New Files (World Pillar — `Scripts/World/UI/`)

| File | Lines (est.) | Purpose |
|------|-------------|---------|
| `HUDManager.cs` | ~60 | Root MonoBehaviour. Owns refs to all sub-components. Handles show/hide. Attached to HUD Canvas. |
| `HealthBarUI.cs` | ~50 | Subscribes to `FloatEventChannel` for health. Animates fill Image. Shows damage flash. |
| `ManaBarUI.cs` | ~45 | Subscribes to `FloatEventChannel` for mana. Animates fill Image. Optional regen tick visual. |
| `ComboCounterUI.cs` | ~70 | Subscribes to `IntEventChannel` for combo hits. Shows count text + decay timer. Punch scale on increment. Fades on combo drop. |
| `PathIndicatorUI.cs` | ~40 | Queries `IPathProvider` on init. Shows Main/Secondary path name text. Empty slots when unselected. |
| `EnemyHealthBarUI.cs` | ~65 | World-space Canvas. Health fill + pressure fill. Reads from parent `EnemyBase`. Auto-destroys with enemy. |

### New Files (Shared)

| File | Lines (est.) | Purpose |
|------|-------------|---------|
| `Shared/Events/FloatEventChannel.cs` | ~25 | SO event channel with `Raise(float value)`. Same pattern as IntEventChannel. |
| `Shared/Components/PlayerManaTracker.cs` | ~50 | Runtime mana tracking. Regen in Update(). Fires FloatEventChannel on change. |

### New Files (Editor)

| File | Lines (est.) | Purpose |
|------|-------------|---------|
| `Editor/HUDCreator.cs` | ~200 | Creates HUD Canvas prefab (screen-space overlay) + EnemyHealthBar prefab (world-space). Creates FloatEventChannel + IntEventChannel SO assets for health/mana/combo. |

### Modified Files

| File | Change |
|------|--------|
| `Combat/PlayerDamageable.cs` | Add `[SerializeField] FloatEventChannel onHealthChanged`. Fire on `TakeDamage()` with normalized health (0-1). |
| `Combat/Combo/ComboController.cs` | Add `[SerializeField] IntEventChannel onComboHitConfirmed`. Fire on `OnHitConfirmed()` with current combo length. |
| `World/EnemyBase.cs` | Add `[SerializeField] GameObject healthBarPrefab`. Instantiate in `Awake()`. Expose pressure fill % as property. |

---

## Acceptance Criteria

- [x] Player health bar (% of max HP) via SO event channel
- [x] Mana bar with regen visualization
- [x] Combo counter with decay timer, subscribes via IntEventChannel
- [x] Path indicator (Main/Secondary icons or text)
- [x] Enemy world-space health bars with pressure meter
- [x] All cross-pillar data flows through SO event channels or Shared interfaces
- [x] No pillar boundary violations
- [x] HUDCreator Editor script builds all prefabs programmatically
- [x] All [SerializeField] injection, no singletons

---

## Execution Order

1. `Shared/Events/FloatEventChannel.cs` — new event channel type
2. `Shared/Components/PlayerManaTracker.cs` — runtime mana tracking
3. Modify `Combat/PlayerDamageable.cs` — fire health changed event
4. Modify `Combat/Combo/ComboController.cs` — fire combo hit confirmed event
5. `World/UI/HealthBarUI.cs` — player health bar
6. `World/UI/ManaBarUI.cs` — player mana bar
7. `World/UI/ComboCounterUI.cs` — combo counter display
8. `World/UI/PathIndicatorUI.cs` — path icons
9. `World/UI/EnemyHealthBarUI.cs` — world-space enemy bars
10. Modify `World/EnemyBase.cs` — spawn health bar prefab
11. `World/UI/HUDManager.cs` — root wiring
12. `Editor/HUDCreator.cs` — Creator Script for all HUD prefabs + SO assets

---

## Pillar Boundary Compliance

```
World Pillar (HUD)
    ├── Subscribes to: FloatEventChannel (SO asset) ← fired by Combat
    ├── Subscribes to: IntEventChannel (SO asset) ← fired by Combat
    ├── Queries: IPathProvider (Shared interface) ← implemented by Roguelite
    ├── Queries: IDamageable (Shared interface) ← on EnemyBase (same pillar)
    └── NEVER imports from: Combat/, Roguelite/, Paths/

Combat Pillar (modifications)
    ├── PlayerDamageable fires: FloatEventChannel (SO asset)
    ├── ComboController fires: IntEventChannel (SO asset)
    └── NEVER imports from: World/, Roguelite/

Shared Layer (new)
    ├── FloatEventChannel.cs — generic, no pillar knowledge
    └── PlayerManaTracker.cs — generic component, no pillar knowledge
```
