# T018: PathSystem — Selection + Tier Progression

**Phase:** 2 | **Priority:** P0 | **Owner:** Dev 2 | **Status:** DONE | **Completed:** 2026-03-03 | **Branch:** pillar2/T018-path-system
**Depends on:** T008 ✓
**Blocks:** T019 (PathSelectionUI), T028 (Path T1 Ability Execution — Dev 1), T041 (InspirationSystem)

---

## Summary

Runtime authority for a player's path state during a run. Enforces Main + Secondary selection rules, tracks tier progression triggered by boss/island defeats, implements `IPathProvider` so all three pillars can query path state, and fires events when selection or tier-up occurs.

---

## Deliverables

| File | Description |
|------|-------------|
| `Assets/Scripts/Paths/PathSystem.cs` | MonoBehaviour implementing IPathProvider |

---

## Acceptance Criteria

- [ ] `SelectMainPath(PathData)` — enforces rules, sets tier 1, fires event; returns bool
- [ ] `SelectSecondaryPath(PathData)` — enforces rules, sets tier 1, fires event; returns bool
- [ ] 3rd path enforced as locked (no `SelectThirdPath` — only two slots exist)
- [ ] `HandleBossDefeated(BossDefeatedData)` — advances both active paths T1→T2
- [ ] `HandleIslandCompleted(IslandCompletedData)` — currently no-op (T3 deferred, max tier is T2)
- [ ] `ResetForNewRun()` — clears all path state (called between runs)
- [ ] `IPathProvider` fully implemented
- [ ] `IRunProgressionEvents` subscribed via SerializeField MonoBehaviour (null-safe)
- [ ] Events fired: `OnMainPathSelected`, `OnSecondaryPathSelected`, `OnPathTierUp`
- [ ] Compiles with zero warnings

---

## Design Decisions

### DD-1: MonoBehaviour

**Decision:** `PathSystem` is a `MonoBehaviour`, not a pure C# class.

**Rationale:** It holds per-run state, participates in Unity's lifecycle for event subscribe/unsubscribe, and needs to be injected into `PathSelectionUI` (T019) and `PathAbilityExecutor` (T028) via `[SerializeField]`. Matches the `CurrencyManager` pattern already in the codebase. The selection/tier logic is simple state management — no testability benefit from splitting into a wrapper + pure C# pair.

### DD-2: Tier-Up via IRunProgressionEvents Subscription (interface-driven)

**Decision:** PathSystem subscribes to `IRunProgressionEvents.OnBossDefeated` and `OnIslandCompleted` using a `[SerializeField] MonoBehaviour` field cast to `IRunProgressionEvents` in `Awake()`. Subscription is null-safe — the field is empty until Dev 3 ships RunManager.

**Rationale:** Keeps pillar coupling through interfaces only. When Dev 3's RunManager lands, PathSystem just gets wired up in the Inspector with zero code changes. Direct `HandleBossDefeated/HandleIslandCompleted` public methods still exist so they can be called manually or via Unity Events in the interim.

```csharp
[SerializeField] private MonoBehaviour _runProgressionSource;

private void Awake()
{
    var src = _runProgressionSource as IRunProgressionEvents;
    if (src != null)
    {
        src.OnBossDefeated    += HandleBossDefeated;
        src.OnIslandCompleted += HandleIslandCompleted;
    }
}

private void OnDestroy()
{
    var src = _runProgressionSource as IRunProgressionEvents;
    if (src != null)
    {
        src.OnBossDefeated    -= HandleBossDefeated;
        src.OnIslandCompleted -= HandleIslandCompleted;
    }
}
```

### DD-3: Selection Returns bool, No Throws

**Decision:** `SelectMainPath` and `SelectSecondaryPath` return `bool` (true = accepted). No exceptions thrown.

**Rationale:** Per coding standards — combat/game code must never throw. `false` is sufficient for the UI to know the selection was rejected; `PathSelectionUI` is responsible for only presenting valid options.

### DD-4: PathSystem Does Not Hold the Available Path List

**Decision:** PathSystem accepts any `PathData` passed to it and validates `data.character == character`. It does NOT maintain a `[SerializeField] PathData[]` of available paths.

**Rationale:** "Which 3 options to show" is a UI concern (`PathSelectionUI`, T019). PathSystem's job is to enforce selection rules, not to curate the menu. This keeps the two responsibilities separate.

### DD-5: Own C# Events (Not IRunProgressionEvents)

**Decision:** PathSystem declares its own `Action<PathSelectedData>` and `Action<PathTierUpData>` events. These are NOT fired through `IRunProgressionEvents`.

**Rationale:** In C#, events can only be raised by the declaring class. Since `IRunProgressionEvents` is implemented by Dev 3's RunManager, PathSystem cannot fire into it. PathSystem uses the same shared data types (`PathSelectedData`, `PathTierUpData`) so subscribers receive identical payloads. If World needs to observe path selection, it subscribes to PathSystem's events via a reference.

---

## Public API

```csharp
public class PathSystem : MonoBehaviour, IPathProvider
{
    // ── Configuration ──────────────────────────────────────────────
    [SerializeField] private CharacterType character;
    [SerializeField] private MonoBehaviour _runProgressionSource; // cast to IRunProgressionEvents

    // ── Events ────────────────────────────────────────────────────
    public event Action<PathSelectedData>  OnMainPathSelected;
    public event Action<PathSelectedData>  OnSecondaryPathSelected;
    public event Action<PathTierUpData>    OnPathTierUp;

    // ── Selection ─────────────────────────────────────────────────
    /// <summary>Select the Main path (T1 granted immediately). Returns false if invalid.</summary>
    public bool SelectMainPath(PathData data);

    /// <summary>Select the Secondary path (T1 granted immediately). Returns false if invalid.</summary>
    public bool SelectSecondaryPath(PathData data);

    // ── Tier Progression (also callable directly for interim testing) ─
    public void HandleBossDefeated(BossDefeatedData data);      // advances T1→T2
    public void HandleIslandCompleted(IslandCompletedData data); // no-op (T3 deferred)

    // ── Run Lifecycle ─────────────────────────────────────────────
    /// <summary>Clears all path state. Call at run start.</summary>
    public void ResetForNewRun();

    // ── IPathProvider ─────────────────────────────────────────────
    public CharacterType Character { get; }
    public PathData MainPath { get; }
    public PathData SecondaryPath { get; }
    public int MainPathTier { get; }
    public int SecondaryPathTier { get; }
    public bool HasPath(PathType type);
    public float GetPathStatBonus(StatType stat);
    public bool IsPathAbilityUnlocked(string abilityId);
}
```

---

## Selection Rules

| Condition | SelectMain | SelectSecondary |
|-----------|-----------|----------------|
| Main already selected | ❌ reject | — |
| Secondary already selected | — | ❌ reject |
| No main selected yet | — | ❌ reject |
| Same path as main | — | ❌ reject |
| `data.character != character` | ❌ reject | ❌ reject |
| `data == null` | ❌ reject | ❌ reject |

---

## Tier Progression Rules

| Event | Main Path | Secondary Path |
|-------|-----------|---------------|
| `HandleBossDefeated` | T1 → T2 (if at T1) | T1 → T2 (if at T1) |
| `HandleIslandCompleted` | no change (T3 deferred) | **no change** |

Tier-up only fires an event if the tier actually changed (idempotent — calling twice at same tier does nothing).

---

## GetPathStatBonus Implementation

```csharp
public float GetPathStatBonus(StatType stat)
{
    float total = 0f;
    if (_mainPath != null && _mainTier > 0)
        total += _mainPath.GetStatBonusArray(_mainTier)[(int)stat];
    if (_secondaryPath != null && _secondaryTier > 0)
        total += _secondaryPath.GetStatBonusArray(_secondaryTier)[(int)stat];
    return total;
}
```

---

## IsPathAbilityUnlocked Implementation

Checks whether an ability ID is unlocked at the current tier on either active path:
```csharp
public bool IsPathAbilityUnlocked(string abilityId)
{
    if (_mainPath != null)
        for (int t = 1; t <= _mainTier; t++)
            if (_mainPath.GetAbilityIdForTier(t) == abilityId) return true;

    if (_secondaryPath != null)
        for (int t = 1; t <= _secondaryTier; t++)
            if (_secondaryPath.GetAbilityIdForTier(t) == abilityId) return true;

    return false;
}
```

---

## Execution Order

1. `PathSystem.cs` — implement everything above

That's it. Single file.
