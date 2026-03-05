# Dump: T021 — RitualSystem Trigger Pipeline (Planning Session)

## Session Info
| Field | Value |
|-------|-------|
| **Task** | T021 — RitualSystem Trigger Pipeline |
| **User** | unknown |
| **Date** | 2026-03-04 |
| **Branch** | gal |
| **Phase** | 2 |
| **Session Type** | `/plan-task` — planning only, no code written |

## Progress

### Task Spec
No spec file exists yet for T021. This session started the planning conversation.
Spec must be written before executing. Run `/plan-task T021` to continue, then `/save-spec` when done.

### Design Decisions Made

#### DD-1: `OnTriggerEffect` shape — **RESOLVED: Option A (data-only struct)**
`OnTriggerEffect` and `OnHitEffect` (both in `Shared/Data/PlaceholderTypes.cs`) will be concrete
data classes with fields Combat reads and applies. No delegates, no enum discriminators.

Proposed shape:
```csharp
public class OnTriggerEffect
{
    public DamageType damageType;      // elemental damage type
    public float damageMultiplier;     // of the hit that triggered it
    public GameObject vfxPrefab;       // spawned at hit position
    public float speedMultiplier;      // temporary speed change (1.0 = none)
}
```
Rationale: keeps Shared as data-only, no cross-pillar callback logic, testable without Unity runtime.

---

#### DD-2: Handler scope in T021 — **RESOLVED: Option A (infrastructure + VFX only)**
T021 builds the trigger pipeline and multiplier caches. Actual DoT damage logic (Burn ticks,
Chain Lightning bounces) is deferred until World pillar IDamageable targets are stable.

Handlers in T021:
- Spawn `effectPrefab` at hit position
- Update multiplier cache (damage/speed/defense)
- DoT tick logic: TODO slot (one-liner addition when World is ready)

Rationale:
- `IDamageable.TakeDamage` in a coroutine requires target lifetime management — World pillar concern
- Multipliers + VFX give immediate ritual impact with zero World dependency
- Adding damage later is a one-liner per handler

---

### Open Design Decisions (Not Yet Discussed)

**DD-3: `OnHitEffect` shape**
Should it mirror `OnTriggerEffect` exactly (same class), or have different fields?
`GetAdditionalOnHitEffects()` fires every hit vs. `GetTriggerEffects()` fires on specific triggers.
Likely same shape, possibly same class.

**DD-4: Multiplier caching strategy**
Recalculate on every query vs. dirty-flag cache updated on `AddRitual`/`LevelUpRitual`.
Recommended: dirty-flag (rituals added infrequently, IBuffProvider queried every combat frame).

**DD-5: ICombatEvents injection pattern**
Same `[SerializeField] private MonoBehaviour _combatEventsSource` pattern as PathSystem.
Likely non-controversial — confirm and move on.

**DD-6: ActiveRitualEntry — nested class vs. separate file**
Runtime wrapper: `RitualData data`, `int level`, `int currentStacks`, `float lastStackTime`.
Likely nested class in RitualSystem.cs — only used by RitualSystem.

**DD-7: Instant ritual damage vs. DoT only**
Some ritual effects could deal damage synchronously in the same frame (not DoT).
e.g., Chain Lightning deals immediate damage to a secondary target.
Decide: can ritual damage be instant, or is all ritual damage scheduled (DoT)?

---

## Planned File Structure

| File | Status |
|------|--------|
| `Scripts/Shared/Data/PlaceholderTypes.cs` | Needs update — flesh out `OnHitEffect` + `OnTriggerEffect` |
| `Scripts/Roguelite/RitualSystem.cs` | Not started |
| `Scripts/Roguelite/ActiveRitualEntry.cs` | Not started (or nested in RitualSystem) |

---

## Blockers / Errors
No blockers. Clean planning session — no errors reported.

## Uncommitted Changes
None — branch `gal` is clean and up to date with origin.

## Context Notes
- T020 (RitualData SO) is merged and on `gal`. Fire + Lightning family assets exist.
- RitualData already references `effectId` as a dispatch table key — architecture aligns.
- PathSystem.cs is the best style reference for MonoBehaviour + interface injection pattern.
- `OnHitEffect` and `OnTriggerEffect` are currently empty placeholder classes in `PlaceholderTypes.cs`.
- No asmdef for Roguelite checked yet — confirm `TomatoFighters.Roguelite.asmdef` exists before writing RitualSystem.

## Recommended Next Steps
1. Run `/plan-task T021` to resume — continue from DD-3 (`OnHitEffect` shape)
2. Resolve remaining DDs (3–7), especially DD-7 (instant vs. DoT) as it affects handler signatures
3. Run `/save-spec` to write the spec to `tasks/phase-2/T021-ritual-system.md`
4. Start a fresh session and run `/task-execute T021`
