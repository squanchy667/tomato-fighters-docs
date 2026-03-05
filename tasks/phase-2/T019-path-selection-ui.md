# T019: PathSelectionUI

| Field       | Value                                    |
|-------------|------------------------------------------|
| Type        | Implementation                           |
| Priority    | P1                                       |
| Owner       | Dev 2                                    |
| Pillar      | Roguelite                                |
| Depends On  | T018 (PathSystem) — DONE                 |
| Blocked By  | None                                     |
| Status      | DONE                                     |
| Completed   | 2026-03-05                               |
| Branch      | `pillar2/T019-path-selection-ui`         |

## Summary

Upgrade shrine UI that lets the player choose their Main path (3 options) or Secondary path (2 options). Displays path name, description, T1 ability preview, and stat bonuses per card. Two-step selection: click to highlight, then confirm. Calls `PathSystem.SelectMainPath()` / `SelectSecondaryPath()` on confirmation.

## Acceptance Criteria

- [ ] Shows all available paths for the character (3 for Main, 2 for Secondary)
- [ ] Displays stat bonuses and T1 ability preview per card
- [ ] Handles both Main and Secondary selection states
- [ ] Greyed-out card for already-selected main path during secondary selection
- [ ] Confirmation button before locking path
- [ ] Pauses game while open, unpauses on close
- [ ] Keyboard shortcuts (1-3) for quick selection
- [ ] Follows OnGUI pattern from CharacterSelectUI

## File Plan

### New Files

| File | Purpose |
|------|---------|
| `Scripts/Roguelite/PathSelectionUI.cs` | The upgrade shrine UI component |
| `Editor/Scenes/PathSelectionTestSceneCreator.cs` | Creator Script for a test scene to verify the UI |

### Files Referenced (read-only)

| File | Usage |
|------|-------|
| `Scripts/Paths/PathSystem.cs` | Injected dependency — `SelectMainPath()` / `SelectSecondaryPath()` |
| `Scripts/Shared/Data/PathData.cs` | Data source for path name, description, bonuses, ability IDs |
| `Scripts/Shared/Data/PathTierBonuses.cs` | Stat bonus fields rendered on each card |
| `Scripts/Shared/Enums/StatType.cs` | Enum for stat labels |
| `Scripts/Characters/CharacterSelectUI.cs` | Pattern reference for OnGUI layout |

## Design Decisions

### DD-1: Inject PathSystem as concrete type (not IPathProvider)

**Rationale:** `SelectMainPath()` and `SelectSecondaryPath()` are on the concrete `PathSystem` class, not on `IPathProvider`. Extending the interface would touch Shared contracts and is out of scope. The UI needs both query (IPathProvider) and mutation (SelectMainPath/SecondaryPath) — concrete injection covers both.

```csharp
[SerializeField] private PathSystem pathSystem;
```

### DD-2: PathData[] injected via SerializeField

**Rationale:** Consistent with project conventions (no singletons, no Resources.Load). The Creator Script or Inspector wires the 3 PathData assets for the character. At runtime, the UI filters based on selection state.

```csharp
[SerializeField] private PathData[] availablePaths; // All 3 paths for this character
```

### DD-3: Two-step confirmation (highlight + CONFIRM button)

**Rationale:** Matches the spec's "Confirmation button" requirement. Click a card (or press 1-3) to highlight it, then click a dedicated CONFIRM button to commit. Prevents accidental path locks. The highlighted card shows a brighter border; unselected cards stay dim.

### DD-4: Public Show() method for triggering

**Rationale:** Keeps the UI decoupled from World pillar events. A future area manager or test scene calls `Show(PathSelectionMode.Main)` or `Show(PathSelectionMode.Secondary)`. The UI doesn't subscribe to `IRunProgressionEvents` directly — that would cross pillar boundaries.

```csharp
public enum PathSelectionMode { Main, Secondary }

public void Show(PathSelectionMode mode)
{
    _currentMode = mode;
    _isSelecting = true;
    _selectedIndex = -1;
    Time.timeScale = 0f;
}
```

### DD-5: Include a test scene Creator Script

**Rationale:** A simple `PathSelectionTestSceneCreator.cs` in `Editor/Scenes/` lets the developer verify the UI without the full game loop. Creates a scene with a PathSystem + PathSelectionUI + the character's 3 PathData assets wired up. Follows the same pattern as `CharacterSelectTestSceneCreator.cs`.

## Implementation Details

### PathSelectionUI.cs — Structure

```csharp
namespace TomatoFighters.Roguelite
{
    public enum PathSelectionMode { Main, Secondary }

    public class PathSelectionUI : MonoBehaviour
    {
        // ── Injected ──
        [SerializeField] private PathSystem pathSystem;
        [SerializeField] private PathData[] availablePaths; // All 3 paths for character

        // ── State ──
        private bool _isSelecting;
        private PathSelectionMode _currentMode;
        private int _selectedIndex = -1;    // -1 = nothing highlighted
        private Texture2D _whiteTexture;

        // ── Public API ──
        public void Show(PathSelectionMode mode) { ... }

        // ── OnGUI rendering ──
        // Dark overlay → Title ("CHOOSE YOUR PATH" / "CHOOSE SECONDARY PATH")
        // → Subtitle with mode context
        // → Path cards (2 or 3 depending on mode)
        //   Each card: path name, description, T1 ability, stat bonuses
        //   Greyed-out card if it's the already-selected main path (secondary mode)
        //   Highlighted border on selected card
        // → CONFIRM button (enabled only when a card is selected)
        // → Keyboard hints

        // ── Selection logic ──
        // SelectPath(int index) → highlights card, sets _selectedIndex
        // ConfirmSelection() → calls pathSystem.SelectMainPath/SecondaryPath
        //   → unpauses, hides self
    }
}
```

### Card Content Per Path

Each card renders:

1. **Path name** — `pathData.pathType.ToString()` (e.g., "Warden", "Bulwark", "Guardian")
2. **Description** — `pathData.description` (TextArea field from SO)
3. **T1 Ability** — Parse `pathData.tier1AbilityId` → split on `_` → show second part (e.g., "Provoke")
4. **Stat bonuses** — Read `pathData.tier1Bonuses` fields, show non-zero values:
   - `+20 HP` / `+5 DEF` / `+0.3 ATK` / etc.
   - Only display stats with non-zero bonuses to keep cards clean

### Color Scheme

Follow CharacterSelectUI pattern:
- Overlay: black at 85% opacity
- Title: gold (`1f, 0.85f, 0.2f`)
- Card backgrounds: path-specific color at 30% opacity (use a color per PathType or a neutral tone)
- Selected card: brighter border (full opacity color)
- Greyed-out card (main path in secondary mode): 20% opacity, no interaction
- CONFIRM button: standard GUI.Button, disabled when `_selectedIndex == -1`

### Stat Bonus Formatting Helper

```csharp
private string FormatStatBonuses(PathTierBonuses bonuses)
{
    var sb = new System.Text.StringBuilder();
    if (bonuses.healthBonus != 0)      sb.AppendLine($"+{bonuses.healthBonus} HP");
    if (bonuses.defenseBonus != 0)     sb.AppendLine($"+{bonuses.defenseBonus} DEF");
    if (bonuses.attackBonus != 0)      sb.AppendLine($"+{bonuses.attackBonus:F1} ATK");
    if (bonuses.rangedAttackBonus != 0) sb.AppendLine($"+{bonuses.rangedAttackBonus:F1} RATK");
    if (bonuses.speedBonus != 0)       sb.AppendLine($"+{bonuses.speedBonus:F1} SPD");
    if (bonuses.manaBonus != 0)        sb.AppendLine($"+{bonuses.manaBonus} MNA");
    if (bonuses.manaRegenBonus != 0)   sb.AppendLine($"+{bonuses.manaRegenBonus:F1} MRG");
    if (bonuses.critChanceBonus != 0)  sb.AppendLine($"+{bonuses.critChanceBonus * 100:F0}% CRIT");
    if (bonuses.stunRateBonus != 0)    sb.AppendLine($"+{bonuses.stunRateBonus:F1} PRS");
    return sb.ToString().TrimEnd();
}
```

### Ability Name Parser

```csharp
private string FormatAbilityName(string abilityId)
{
    if (string.IsNullOrEmpty(abilityId)) return "—";
    int underscore = abilityId.IndexOf('_');
    return underscore >= 0 ? abilityId.Substring(underscore + 1) : abilityId;
}
```

### GetDisplayPaths() Logic

```csharp
private PathData[] GetDisplayPaths()
{
    if (_currentMode == PathSelectionMode.Main)
        return availablePaths; // all 3

    // Secondary: exclude already-selected main path
    return System.Array.FindAll(availablePaths, p => p != pathSystem.MainPath);
}
```

## Out of Scope

- RitualSystem integration (separate task)
- Stat recalculation preview (before/after comparison) — CharacterStatCalculator wiring is downstream
- PathAbilityExecutor wiring — T028 handles this independently
- UGUI/Canvas conversion — OnGUI is the established pattern for now
- Animated transitions or DOTween juice on the UI

## Test Scene Creator

`Editor/Scenes/PathSelectionTestSceneCreator.cs` creates a minimal scene:
- Camera
- PathSystem component (set to a test character, e.g., Brutor)
- PathSelectionUI component with `availablePaths` wired to the 3 Brutor PathData assets
- Auto-calls `Show(PathSelectionMode.Main)` on Start via a small test bootstrap script
- Menu item: `TomatoFighters/Scenes/Create Path Selection Test Scene`
