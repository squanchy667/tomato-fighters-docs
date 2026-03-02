# Dev 1 Guide — Combat Sandbox

**Owner:** Dev 1
**Pillar:** Combat Sandbox + Character Movesets
**Branch prefix:** `pillar1/combat-*`
**AgentPilot config:** `unity-combat`, `unity-combat-feel`
**Context strategy:** `combat` (sees: `Combat/`, `Characters/`, `Shared/`)

---

## Your Domain

You own everything about how combat FEELS and WORKS at the frame level:

| System | Key Files | What It Does |
|--------|----------|-------------|
| Character Controller | `Combat/CharacterController2D.cs` | Movement, jump, dash per character |
| Input Buffer | `Combat/InputBufferSystem.cs` | 6-frame pre-buffering |
| Combo State Machine | `Combat/ComboSystem.cs`, `ComboNode.cs` | Per-character combo trees |
| Hitbox Manager | `Combat/HitboxManager.cs` | Animation Event-driven hitboxes |
| Defense System | `Combat/DefenseSystem.cs` | Deflect/Clash/Dodge per character |
| Pressure/Stun | `Combat/PressureSystem.cs` | Pressure meter, stun state |
| Wall Bounce | `Combat/WallBounceHandler.cs` | Wall collision physics |
| Air Juggle | `Combat/JuggleSystem.cs` | Airborne tracking, OTG/Tech |
| Anti-Spam | `Combat/RepetitiveTracker.cs` | Damage penalty for spam |
| Path Abilities | `Characters/PathAbilityExecutor.cs` | Execute all 12×3 tier abilities |
| Character Passives | `Characters/PassiveAbilitySystem.cs` | 4 passive abilities |

## You Do NOT Touch

- `Roguelite/` — Dev 2's domain
- `Paths/` — Dev 2's domain (you QUERY via IPathProvider, never write)
- `World/` — Dev 3's domain
- `Animations/` — Dev 3 sets up controllers, you define the state requirements

## How You Communicate with Other Pillars

| Direction | Interface | What Happens |
|-----------|-----------|-------------|
| You → Dev 2 | `ICombatEvents` | You **fire** events (OnStrike, OnDeflect, etc.) → Dev 2's RitualSystem subscribes |
| Dev 2 → You | `IBuffProvider` | You **query** damage/speed/defense multipliers from Dev 2's buff system |
| Dev 2 → You | `IPathProvider` | You **query** which paths are active, what tier, which abilities unlocked |
| You → Dev 3 | `IDamageable` | You **call** TakeDamage on enemies through the interface |
| Dev 3 → You | `IAttacker` | You **read** enemy attack data to determine deflect/clash/dodge response |

## Your Task Sequence

```
Phase 1: T002 → T003 → T004 → T005 (controller, input, combo, attackdata)
Phase 2: T014 → T015 → T016, T017 (combo all chars, hitbox, defense, passives)
Phase 3: T026, T027, T028 (pressure, wallbounce, path T1 abilities)
Phase 4: T035, T036, T037 (anti-spam, OTG, path T2+T3 abilities)
Phase 5: T045, T046 (feel pass, arcana)
Phase 6: T053, T054 (game feel, combat balance)
```

## Running a Task

```bash
# From the project root, use the existing meta-commands:
/do-task "T002: CharacterController2D — 8-dir movement, jump, dash for Brutor with armored dash. Uses Rigidbody2D, all values configurable in Inspector."

# Or for a full phase:
/execute-phase 1
```

## Character Combat Differences to Remember

| Character | Strike Speed | Dash Type | Deflect Bonus | Clash Bonus |
|-----------|-------------|-----------|---------------|-------------|
| Brutor | Slow | Armored (super-armor) | No slideback | Extra knockback |
| Slasher | Fast (4-hit) | Through-enemies | Guaranteed crit | Free dash-cancel |
| Mystica | Medium (projectile) | Blink teleport | Restore 5% mana | Free next cast |
| Viper | Medium (ranged) | Backflip + shot | Projectile reflect | Drop caltrops |
