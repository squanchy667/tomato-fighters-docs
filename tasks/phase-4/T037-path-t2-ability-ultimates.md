# T037: Path T2 Passive Enhancements + Character Ultimates

> **Phase:** 4 | **Priority:** P0 | **Owner:** Dev 1 | **Depends on:** T028
> **Status:** PLANNING

---

## Summary

Implement 12 T2 passive ability enhancements (one per path), 4 character Ultimates, and update PathAbilityExecutor to support both systems. T3 abilities are deferred — not in scope.

### Revised Ability Model

| Slot | Source | Input | Notes |
|------|--------|-------|-------|
| **Special 1** | Main path T1 active | Q | Already exists from T028 |
| **Special 2** | Secondary path T1 active | E | Already exists from T028 |
| **T2 Passive** | Auto-applied at tier-up | None | Enhances gameplay, no input needed |
| **Ultimate** | Character-specific, fixed | Hold R (or dedicated button) | 1 charge per run, rare to earn more |

- **Only 2 active abilities** (Q and E) — everything else is passive or charge-based
- **No T3 abilities** — tier 3 is deferred
- **Ultimates are character-fixed** — not affected by path selection

---

## Deliverable 1: Four Character Ultimates

### Brutor — "Unbreakable"

Brutor plants his feet and becomes fully invulnerable for 6 seconds. All enemies within 8 units are force-taunted onto him. Incoming attacks reflect 40% damage back to attackers. When the duration ends, Brutor releases a ground slam dealing 150% of total absorbed damage (min 100, max 800) in a 5-unit radius. Brutor cannot move during Unbreakable.

| Property | Value |
|----------|-------|
| Duration | 6 seconds (rooted) |
| Taunt radius | 8 units |
| Damage reflect | 40% of incoming |
| End slam | 150% absorbed (min 100, max 800) |
| Charges | 1 per run |

### Slasher — "Phantom Strike"

Slasher dashes forward across the entire screen in a straight line, cutting through every enemy. Deals 800% ATK to all enemies hit. If any enemy dies from the strike, Slasher reappears at their position and can dash again (up to 3 resets). Slasher is invulnerable during each dash. Each reset targets the nearest surviving enemy's direction.

| Property | Value |
|----------|-------|
| Dash distance | Full screen (~20 units) |
| Damage | 800% ATK per enemy hit |
| Kill resets | Up to 3 additional dashes |
| Invulnerable | During each dash |
| Charges | 1 per run |

### Mystica — "Time Warp"

Mystica freezes all enemies on screen for 4 seconds. During the freeze, allies move and attack freely. All damage dealt during the freeze is stored and applied at once when time resumes, with a 50% bonus. Additionally, any downed allies are instantly revived at 50% HP.

| Property | Value |
|----------|-------|
| Freeze duration | 4 seconds |
| Stored damage bonus | +50% on release |
| Revive | All downed allies at 50% HP |
| Charges | 1 per run |

### Viper — "Rain of Arrows"

Viper marks a large target area. After 1 second, a barrage rains down for 3 seconds — 10 hits total, each dealing 100% RATK to all enemies in the zone. Enemies inside are slowed 50%. Viper can move freely during the barrage.

| Property | Value |
|----------|-------|
| Target area radius | 4 units |
| Delay | 1 second |
| Barrage duration | 3 seconds (10 hits) |
| Damage per hit | 100% RATK |
| Slow | 50% inside zone |
| Charges | 1 per run |

---

## Deliverable 2: Twelve T2 Passive Enhancements

T2 passives auto-activate when the player reaches Tier 2 on a path (after defeating the Area Boss). They are **not** input-activated — they modify existing mechanics passively.

### Brutor Paths

| Path | T2 Passive | Description |
|------|-----------|-------------|
| **Warden** | Aggro Aura | While 3+ enemies are targeting Brutor, gain +25% ATK and +15% SPD |
| **Bulwark** | Retaliation | Every 3rd hit blocked triggers an auto counter-strike: 150% ATK, ignores enemy DEF, staggers |
| **Guardian** | Rallying Presence | Allies within medium radius gain +10 DEF and regenerate 2% max HP/s. Brutor heals 1% per ally in range |

### Slasher Paths

| Path | T2 Passive | Description |
|------|-----------|-------------|
| **Executioner** | Execution Threshold | Enemies below 30% HP take +50% damage from Slasher. Crits against low-HP enemies deal 2.5x (up from 1.5x) |
| **Reaper** | Chain Slash | On kill, automatically dash to nearest enemy within 4 units and deal 80% ATK. Resets once per kill |
| **Shadow** | Afterimage | After dashing, leave an afterimage (1.5s). If an enemy attacks it, Slasher appears behind that enemy and deals 120% ATK guaranteed crit. 4s internal cooldown |

### Mystica Paths

| Path | T2 Passive | Description |
|------|-----------|-------------|
| **Sage** | Purifying Presence | Mending Aura also cleanses one negative status effect per tick from each ally in range |
| **Enchanter** | Elemental Infusion | Empower also infuses the target's attacks with the dominant ritual element for the buff duration |
| **Conjurer** | Totem Pulse | Sproutlings pulse AoE damage (50% ATK) and 20% slow in a 2-unit radius every 2 seconds |

### Viper Paths

| Path | T2 Passive | Description |
|------|-----------|-------------|
| **Marksman** | Rapid Volleys | Every 5th ranged attack fires a 3-shot burst (60% RATK each). Combined with Piercing, shreds groups |
| **Trapper** | Trap Deployment | Harpoon Shot also deploys an invisible trap at the impact point. Trap snares first enemy to walk over it for 2s |
| **Arcanist** | Mana Blast | Releasing Mana Charge fires a piercing beam instead of restoring mana. Damage scales with charge: 25%=100% ATK, 50%=250%, 75%=400%, 100%=600% ATK |

---

## Deliverable 3: Ultimate Charge System

### UltimateController

New MonoBehaviour living on the player prefab alongside PathAbilityExecutor.

```csharp
public class UltimateController : MonoBehaviour
{
    [SerializeField] private int maxCharges = 1;

    private int _currentCharges;
    private IUltimate _ultimate;  // Set based on CharacterType

    public int CurrentCharges => _currentCharges;
    public bool CanActivate => _currentCharges > 0 && !_ultimate.IsActive;

    public void Initialize(CharacterType character) { /* create from factory */ }
    public void TryActivate() { /* consume charge, fire ultimate */ }
    public void AddCharge() { /* rare reward grants +1 charge */ }
}
```

- Separate from PathAbilityExecutor — Ultimates are character-fixed, not path-dependent
- Input: hold R (or dedicated button) routed from CharacterInputHandler
- Charges persist for the run, reset between runs

### IUltimate Interface

```csharp
public interface IUltimate
{
    string UltimateId { get; }
    bool IsActive { get; }
    bool TryActivate(UltimateContext context);
    void Tick(float deltaTime);
    void Cleanup();
}
```

Separate from IPathAbility — Ultimates are not path abilities, they don't have cooldowns or mana costs, they use charges.

---

## Deliverable 4: PathAbilityExecutor Changes

### Add T2 passive support

When a path tiers up to T2, the executor creates the T2 passive and registers it:

```csharp
public void OnTierUp(bool isMain, int newTier)
{
    if (newTier != 2) return; // Only T2 passives for now

    var path = isMain ? _mainPath : _secondaryPath;
    string abilityId = path.tier2AbilityId;
    var passive = AbilityFactory.Create(abilityId, _context);
    if (passive == null) return;

    _t2Passives.Add(passive);
    RegisterModifier(passive);
    passive.TryActivate();
    _activeAbilities.Add(passive);
}
```

### Remove T3 logic

No T3 ability creation or routing. PathData still has tier3AbilityId field (for future use) but the executor ignores it.

---

## File Plan

### New Files (20 total)

| # | File | Description |
|---|------|-------------|
| 1 | `Shared/Interfaces/IUltimate.cs` | Ultimate interface (separate from IPathAbility) |
| 2 | `Characters/UltimateController.cs` | MonoBehaviour: charge management, input routing |
| 3 | `Characters/Abilities/Ultimates/UltimateContext.cs` | Dependency bundle for ultimates |
| 4 | `Characters/Abilities/Ultimates/UltimateFactory.cs` | CharacterType -> IUltimate mapping |
| 5 | `Characters/Abilities/Ultimates/BrutorUnbreakable.cs` | 6s invuln + taunt + reflect + slam |
| 6 | `Characters/Abilities/Ultimates/SlasherPhantomStrike.cs` | Screen dash + kill resets |
| 7 | `Characters/Abilities/Ultimates/MysticaTimeWarp.cs` | Enemy freeze + stored damage + revive |
| 8 | `Characters/Abilities/Ultimates/ViperRainOfArrows.cs` | AoE barrage zone |
| 9 | `Characters/Abilities/Warden/AggroAura.cs` | T2 passive: conditional ATK/SPD buff |
| 10 | `Characters/Abilities/Bulwark/Retaliation.cs` | T2 passive: auto counter on 3rd block |
| 11 | `Characters/Abilities/Guardian/RallyingPresence.cs` | T2 passive: aura heal + DEF |
| 12 | `Characters/Abilities/Executioner/ExecutionThreshold.cs` | T2 passive: bonus damage on low-HP |
| 13 | `Characters/Abilities/Reaper/ChainSlash.cs` | T2 passive: auto-dash on kill |
| 14 | `Characters/Abilities/Shadow/Afterimage.cs` | T2 passive: dash afterimage bait |
| 15 | `Characters/Abilities/Sage/PurifyingPresence.cs` | T2 passive: aura cleanse |
| 16 | `Characters/Abilities/Enchanter/ElementalInfusion.cs` | T2 passive: empower adds element |
| 17 | `Characters/Abilities/Conjurer/TotemPulse.cs` | T2 passive: sproutlings pulse AoE |
| 18 | `Characters/Abilities/Marksman/RapidVolleys.cs` | T2 passive: every 5th shot bursts |
| 19 | `Characters/Abilities/Trapper/TrapDeployment.cs` | T2 passive: harpoon drops trap |
| 20 | `Characters/Abilities/Arcanist/ManaBlast.cs` | T2 passive: charge release fires beam |

### Modified Files (3 total)

| # | File | Changes |
|---|------|---------|
| 1 | `Characters/PathAbilityExecutor.cs` | Add `_t2Passives` list, `OnTierUp()` method, remove T3 slot logic |
| 2 | `Characters/Abilities/AbilityFactory.cs` | Register all 12 T2 passive IDs |
| 3 | `Characters/CharacterInputHandler.cs` | Route hold-R input to UltimateController |

### No changes to Editor/Creator scripts

PathDataCreator already has T2 ability IDs defined. PathData SO structure unchanged.

---

## Execution Order

1. **IUltimate interface + UltimateContext** — shared contracts first
2. **UltimateFactory + UltimateController** — charge system framework
3. **4 Ultimate classes** — one per character
4. **12 T2 passive classes** — one per path
5. **AbilityFactory updates** — register T2 ability IDs
6. **PathAbilityExecutor updates** — OnTierUp() + T2 passive management
7. **CharacterInputHandler** — hold-R routing to UltimateController

---

## Acceptance Criteria

- [ ] All 12 T2 passives functional and auto-activate at tier-up
- [ ] All 4 Ultimates functional with charge consumption
- [ ] Ultimate charges start at 1, AddCharge() works for rare rewards
- [ ] Hold-R (or dedicated button) activates Ultimate
- [ ] T2 passives registered as IPathAbilityModifier where applicable
- [ ] No T3 abilities created or routed
- [ ] Compiles with zero warnings

---

## Design Decisions

### DD-1: Separate Ultimate system from PathAbilityExecutor

Ultimates are character-fixed, not path-dependent. They use charges instead of cooldowns/mana. Keeping them in a separate UltimateController avoids overloading PathAbilityExecutor with mixed concerns.

### DD-2: T2 abilities are ALL passive

No new active abilities at T2. This keeps the input model clean: Q = special 1, E = special 2, hold R = ultimate. T2 passives enhance existing mechanics without adding button complexity.

### DD-3: Originally-active T2 abilities redesigned as passives

Several T2 abilities from CHARACTER-ARCHETYPES.md were originally active (Chain Slash, Purifying Burst, etc.). These have been reworked as passive triggers or enhancements:
- Chain Slash -> auto-dash on kill
- Purifying Burst -> Mending Aura also cleanses
- Elemental Infusion -> Empower also infuses element
- Deploy Totem -> Sproutlings pulse AoE
- Rapid Fire -> every 5th shot bursts
- Trap Net -> Harpoon also deploys trap
- Mana Blast -> charge release fires beam (replaces mana restore)

### DD-4: IUltimate is a separate interface from IPathAbility

Ultimates don't have cooldowns, mana costs, or activation types. They consume charges. A separate interface avoids forcing square pegs into round holes.

### DD-5: No T3 — deferred

T3 signature abilities are not implemented. PathData still holds tier3AbilityId for future use, but the executor ignores it. This keeps scope manageable and the ability model flat.
