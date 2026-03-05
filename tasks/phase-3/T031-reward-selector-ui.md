# T031: RewardSelectorUI

> **Status:** DONE | **Priority:** P1 | **Owner:** Dev 2 | **Depends on:** T021
> **Pillar:** Roguelite | **Phase:** 3 | **Branch:** `roguelite/T031-reward-selector-ui` | **Completed:** 2026-03-05

---

## Summary

Post-area reward selection screen. When the player clears a combat area, gameplay pauses and the UI offers 2 ritual options (3 with Soul Tree upgrade), plus gold and crystal currency alternatives. Each ritual card shows name, family (color-coded), description, trigger, and effect preview. Player picks one, selection is applied, event fires, gameplay resumes.

## Why It Matters

This is the primary roguelite loop driver. Every area clear cycles through this screen. Without it:
- RitualSystem has no way to gain rituals during a run
- CurrencyManager has no source of run income
- The game loop stalls after combat
- T038 (SoulTree) depends on the "3rd ritual choice" hook

---

## File Plan

| # | File | Type | Purpose |
|---|------|------|---------|
| 1 | `Scripts/Shared/Data/RewardSelectedData.cs` | Struct | Carries selection result (ritual or currency) |
| 2 | `Scripts/Shared/Events/RewardSelectedEventChannel.cs` | SO event channel | Fires RewardSelectedData when player picks a reward |
| 3 | `Scripts/Roguelite/RewardOption.cs` | Plain C# class | Uniform display model wrapping RitualData or currency reward |
| 4 | `Scripts/Roguelite/RitualPoolSelector.cs` | Plain C# class | Picks N random rituals from available pool, weighted by category, filtering maxed |
| 5 | `Scripts/Roguelite/RewardConfig.cs` | ScriptableObject | Configurable currency amounts per area, category weights, pool settings |
| 6 | `Scripts/Roguelite/RewardSelectorUI.cs` | MonoBehaviour | Main UI controller — generation, display, selection, event firing |
| 7 | `Editor/Prefabs/RewardSelectorUICreator.cs` | Editor Creator Script | Builds UI prefab + wires SO event channels |
| 8 | `Tests/EditMode/Roguelite/RitualPoolSelectorTests.cs` | NUnit test | Tests ritual selection, filtering, weighting |

---

## Design Decisions

### DD-1: Separate RitualPoolSelector (plain C# class)
**Decision:** Extract ritual selection logic into a standalone `RitualPoolSelector` class.
**Rationale:** Testable in EditMode without MonoBehaviour. WaveManager/BossAI could reuse for different reward pools. Clean separation: UI displays, selector picks.

```csharp
public class RitualPoolSelector
{
    /// <summary>
    /// Selects ritual options from the available pool, filtering out maxed rituals.
    /// </summary>
    public List<RitualData> SelectOptions(
        RitualData[] availablePool,
        List<ActiveRitualInfo> activeRituals,
        int count,
        float[] categoryWeights)
    {
        // 1. Filter out rituals already at max level (3)
        // 2. Weight by category: Core=1.0, General=0.8, Enhancement=0.5, Twin=0.3
        // 3. Pick `count` unique options via weighted random
        // 4. Fallback: if pool exhausted, return fewer options
    }
}

/// <summary>Lightweight snapshot of an active ritual for pool filtering.</summary>
public struct ActiveRitualInfo
{
    public string ritualName;
    public int currentLevel;
}
```

### DD-2: SO event channel trigger (decoupled from World pillar)
**Decision:** Use an intermediate `VoidEventChannel` SO to trigger the reward screen, rather than direct subscription to `IRunProgressionEvents`.
**Rationale:** Keeps the UI fully decoupled from the World pillar. The scene wiring (or a mediator MonoBehaviour) bridges `OnAreaCleared` → `VoidEventChannel`. RewardSelectorUI only knows about the SO channel.

```csharp
// RewardSelectorUI subscribes to this, not to IRunProgressionEvents directly
[SerializeField] private VoidEventChannel onShowRewardSelector;

private void OnEnable() => onShowRewardSelector.Register(ShowRewards);
private void OnDisable() => onShowRewardSelector.Unregister(ShowRewards);
```

### DD-3: Configurable currency amounts via RewardConfig SO
**Decision:** Create a `RewardConfig` ScriptableObject for all tunable reward parameters.
**Rationale:** Designers can tweak amounts in Inspector without code changes. Supports future scaling per area/island.

```csharp
[CreateAssetMenu(menuName = "TomatoFighters/RewardConfig")]
public class RewardConfig : ScriptableObject
{
    [Header("Ritual Options")]
    [Range(2, 3)] public int baseRitualChoices = 2;
    public float[] categoryWeights = { 1.0f, 0.8f, 0.5f, 0.3f }; // Core, General, Enhancement, Twin

    [Header("Currency Alternatives")]
    public int crystalRewardAmount = 50;
    public int imbuedFruitRewardAmount = 1;

    [Header("Family Colors")]
    public Color fireColor = new Color(1f, 0.3f, 0f);
    public Color lightningColor = new Color(0.8f, 0.8f, 0f);
    public Color waterColor = new Color(0f, 0.5f, 1f);
    public Color thornColor = new Color(0.2f, 0.8f, 0.2f);
    public Color galeColor = new Color(0.7f, 0.9f, 1f);
    public Color timeColor = new Color(0.6f, 0.4f, 0.8f);
    public Color cosmicColor = new Color(0.9f, 0.2f, 0.9f);
    public Color necroColor = new Color(0.3f, 0f, 0.3f);

    public Color GetFamilyColor(RitualFamily family) { /* switch on family */ }
}
```

### DD-4: Time.timeScale = 0 pause during reward screen
**Decision:** Pause gameplay with `Time.timeScale = 0` while the reward UI is active.
**Rationale:** Standard for this genre. Prevents enemies from acting, rituals from ticking, physics from running during selection. All UI animations use `unscaledDeltaTime` / `WaitForSecondsRealtime`.

### DD-5: Typed RewardSelectedEventChannel (SO pattern)
**Decision:** Create a dedicated `RewardSelectedEventChannel` SO event channel.
**Rationale:** Consistent with existing VoidEventChannel/IntEventChannel/FloatEventChannel pattern. RitualSystem and CurrencyManager subscribe via [SerializeField] injection.

```csharp
[CreateAssetMenu(fileName = "NewRewardSelectedEvent",
                 menuName = "TomatoFighters/Events/Reward Selected Event Channel")]
public class RewardSelectedEventChannel : ScriptableObject
{
    private Action<RewardSelectedData> _onRaised;
    public void Register(Action<RewardSelectedData> listener) => _onRaised += listener;
    public void Unregister(Action<RewardSelectedData> listener) => _onRaised -= listener;
    public void Raise(RewardSelectedData data) => _onRaised?.Invoke(data);
}
```

---

## Data Structures

### RewardSelectedData (Shared/Data/)
```csharp
public struct RewardSelectedData
{
    public RewardType rewardType;       // Ritual, Crystals, ImbuedFruit
    public RitualData selectedRitual;   // null if currency
    public CurrencyType currencyType;   // only if rewardType != Ritual
    public int currencyAmount;          // only if rewardType != Ritual
}

public enum RewardType { Ritual, Currency }
```

### RewardOption (Roguelite/)
```csharp
/// <summary>Uniform display model for one reward card.</summary>
public class RewardOption
{
    public RewardType type;
    public string displayName;
    public string description;
    public Color borderColor;
    public RitualData ritualData;       // null for currency options
    public CurrencyType currencyType;
    public int currencyAmount;

    public static RewardOption FromRitual(RitualData data, RewardConfig config) { /* ... */ }
    public static RewardOption FromCurrency(CurrencyType type, int amount) { /* ... */ }
}
```

---

## RewardSelectorUI Architecture

```csharp
public class RewardSelectorUI : MonoBehaviour
{
    [Header("Config")]
    [SerializeField] private RewardConfig config;
    [SerializeField] private RitualData[] ritualPool;      // all available rituals

    [Header("Dependencies")]
    [SerializeField] private Component ritualSystemComponent;  // cast to RitualSystem at runtime
    [SerializeField] private CurrencyManager currencyManager;

    [Header("Events")]
    [SerializeField] private VoidEventChannel onShowRewardSelector;
    [SerializeField] private RewardSelectedEventChannel onRewardSelected;

    [Header("Soul Tree Hook")]
    [Tooltip("Set to true when Soul Tree '3rd Choice' is unlocked")]
    [SerializeField] private bool hasThirdChoice = false;

    [Header("UI References")]
    [SerializeField] private GameObject rootPanel;
    [SerializeField] private Transform optionsContainer;
    [SerializeField] private GameObject rewardCardPrefab;

    private RitualPoolSelector _poolSelector;
    private List<RewardOption> _currentOptions;

    private void Awake()
    {
        _poolSelector = new RitualPoolSelector();
        rootPanel.SetActive(false);
    }

    private void OnEnable() => onShowRewardSelector.Register(ShowRewards);
    private void OnDisable() => onShowRewardSelector.Unregister(ShowRewards);

    private void ShowRewards()
    {
        // 1. Pause
        Time.timeScale = 0f;

        // 2. Generate ritual options
        int ritualCount = hasThirdChoice ? 3 : config.baseRitualChoices;
        var activeRituals = GetActiveRitualInfo();
        var rituals = _poolSelector.SelectOptions(ritualPool, activeRituals, ritualCount, config.categoryWeights);

        // 3. Build display options (rituals + currency alternatives)
        _currentOptions = new List<RewardOption>();
        foreach (var r in rituals)
            _currentOptions.Add(RewardOption.FromRitual(r, config));
        _currentOptions.Add(RewardOption.FromCurrency(CurrencyType.Crystals, config.crystalRewardAmount));
        _currentOptions.Add(RewardOption.FromCurrency(CurrencyType.ImbuedFruits, config.imbuedFruitRewardAmount));

        // 4. Populate UI cards
        PopulateCards(_currentOptions);

        // 5. Show panel
        rootPanel.SetActive(true);
    }

    public void OnOptionSelected(int index)
    {
        var option = _currentOptions[index];

        // Apply selection
        if (option.type == RewardType.Ritual)
        {
            var ritualSystem = ritualSystemComponent as RitualSystem;  // safe within Roguelite pillar
            ritualSystem.AddRitual(option.ritualData);
        }
        else
        {
            currencyManager.TryAdd(option.currencyType, option.currencyAmount);
        }

        // Fire event
        onRewardSelected.Raise(new RewardSelectedData
        {
            rewardType = option.type,
            selectedRitual = option.ritualData,
            currencyType = option.currencyType,
            currencyAmount = option.currencyAmount
        });

        // Cleanup
        rootPanel.SetActive(false);
        Time.timeScale = 1f;
    }
}
```

---

## UI Prefab Structure (built by Creator Script)

```
RewardSelectorUI (Canvas, ScreenSpace-Overlay)
├── Background (Image, black 60% alpha, covers screen)
├── Panel (centered, ~60% screen width)
│   ├── Title (TextMeshProUGUI, "Choose Your Reward")
│   ├── OptionsContainer (HorizontalLayoutGroup)
│   │   ├── RewardCard_0 (ritual)
│   │   │   ├── BorderImage (colored by family)
│   │   │   ├── FamilyIcon (Image)
│   │   │   ├── NameText (TMP)
│   │   │   ├── DescriptionText (TMP)
│   │   │   ├── TriggerText (TMP, "Triggers on: Strike")
│   │   │   └── SelectButton (Button)
│   │   ├── RewardCard_1 (ritual)
│   │   ├── RewardCard_2 (ritual, hidden if !hasThirdChoice)
│   │   ├── Divider (Image, vertical line)
│   │   ├── CurrencyCard_Crystals
│   │   │   ├── CrystalIcon
│   │   │   ├── AmountText
│   │   │   └── SelectButton
│   │   └── CurrencyCard_ImbuedFruit
│   │       ├── FruitIcon
│   │       ├── AmountText
│   │       └── SelectButton
│   └── HintText (TMP, "Pick one reward")
└── RewardSelectorUI (component, wired to all refs)
```

---

## Execution Order

1. `Shared/Data/RewardSelectedData.cs` + `RewardType` enum
2. `Shared/Events/RewardSelectedEventChannel.cs`
3. `Roguelite/RewardOption.cs`
4. `Roguelite/RewardConfig.cs`
5. `Roguelite/RitualPoolSelector.cs`
6. `Roguelite/RewardSelectorUI.cs`
7. `Editor/Prefabs/RewardSelectorUICreator.cs`
8. `Tests/EditMode/Roguelite/RitualPoolSelectorTests.cs`

---

## Acceptance Criteria

- [ ] 2-3 ritual options displayed (3 when `hasThirdChoice = true`)
- [ ] Gold and crystal alternatives shown
- [ ] Ritual preview with family color coding
- [ ] Selection fires `RewardSelectedEventChannel` consumed by RitualSystem
- [ ] Gameplay paused during selection (timeScale = 0)
- [ ] RitualPoolSelector filters out maxed rituals
- [ ] RitualPoolSelector weighted by category
- [ ] Creator Script builds UI prefab + wires SO channels
- [ ] EditMode tests for RitualPoolSelector

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Ritual pool empty (no SOs created yet) | RitualDataCreator.cs exists; if pool is empty, show only currency options gracefully |
| Soul Tree not built (T038) | `hasThirdChoice` defaults false; SoulTree sets it when implemented |
| Input conflict during reward screen | Disable combat action map, enable UI action map via InputSystem |
| Time.timeScale = 0 freezes coroutines | All UI animations use `WaitForSecondsRealtime` and `unscaledDeltaTime` |
| Level-up existing ritual (not just add new) | RitualSystem.AddRitual already handles level-up if ritual exists |
