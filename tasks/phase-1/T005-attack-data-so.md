# T005: AttackData ScriptableObject

## Metadata
| Field | Value |
|-------|-------|
| **Phase** | 1 — Foundation |
| **Type** | implementation |
| **Priority** | P0 |
| **Owner** | Dev 1 |
| **Agent** | so-architect |
| **Depends On** | T001 |
| **Blocks** | T014, T023 |
| **Status** | DONE |
| **Completed** | 2026-03-03 |
| **Branch** | `pillar1/T005-attack-data-so` |

## Objective
Define the AttackData ScriptableObject — the universal data container for every attack in the game — and create Mystica's initial 4-attack set as concrete asset instances. This SO is referenced by ComboNode, HitboxManager, EnemyAI, and the damage pipeline, making it one of the most-referenced data types in the project.

## Context
Every attack in Tomato Fighters — player strikes, enemy swings, boss slams, path abilities — is defined by an AttackData ScriptableObject. This is the data-driven core of the combat system: no attack parameters are hardcoded in scripts.

The AttackData definition is specified in `architecture/interface-contracts.md` under "Shared Data Structures". It lives in `Shared/Data/` because both Combat (reads it for player attacks) and World (reads it for enemy attacks via IAttacker) reference it.

For Phase 1, this task creates:
1. The AttackData class definition
2. Mystica's 4 attack assets: Strike 1 (Magic Burst 1), Strike 2 (Magic Burst 2), Strike 3 (Magic Burst 3), Heavy (Arcane Bolt)

These assets will be referenced by ComboNode instances in T004 (combo chain) and later by HitboxManager in T015.

From the character spec, Mystica's base ATK is 0.5 (lowest) and her strikes are described as "3-hit magical projectile burst (short range, piercing). Low damage but safe." Her heavy is "Arcane bolt — medium range, homing, moderate damage."

## Requirements

### AttackData ScriptableObject Definition

1. **Core damage fields:**
   - `float damageMultiplier` — multiplied by character's base ATK and all buff multipliers to get final damage. Example: Brutor ATK 0.7 × damageMultiplier 1.2 = 0.84 base damage before buffs
   - `Vector2 knockbackForce` — applied to target via Rigidbody2D on hit. X = horizontal push, Y = upward component
   - `Vector2 launchForce` — separate from knockback; used for launchers that send enemies airborne. Zero for non-launchers
   - `bool causesWallBounce` — if true and target hits a wall during knockback, triggers wall bounce (WallBounceHandler, T027)
   - `bool causesLaunch` — if true, target enters airborne state (JuggleSystem, T027)

2. **Animation and timing fields:**
   - `AnimationClip animationClip` — reference to the animation this attack plays (can be null for placeholder; Animator Override Controller in T024 will map these)
   - `int hitboxStartFrame` — frame number when the hitbox activates (0-indexed from animation start)
   - `int hitboxActiveFrames` — number of frames the hitbox stays active
   - `int totalFrames` — total animation length in frames (for combo window timing)
   - `float animationSpeed` — playback speed multiplier (default 1.0; Brutor attacks are slower, Slasher faster)

3. **Telegraph and state fields:**
   - `TelegraphType telegraphType` — `Normal` or `Unstoppable`. Normal attacks can be deflected/clashed. Unstoppable (red flash) bypasses deflect and clash — only dodge avoids them
   - `bool isOTGCapable` — can hit downed/grounded enemies (Brutor's overhead slam)
   - `bool isAirAttack` — can be used while airborne

4. **Combo integration fields:**
   - `string attackId` — unique identifier (e.g., "brutor_strike_1", "brutor_finisher")
   - `string attackName` — display name for UI/debug ("Shield Jab", "Overhead Slam")

5. **VFX/SFX references (for future use):**
   - `GameObject hitEffectPrefab` — particle/VFX spawned on hit (can be null in Phase 1)
   - `AudioClip swingSound` — sound on attack start (can be null in Phase 1)
   - `AudioClip hitSound` — sound on hit confirm (can be null in Phase 1)

6. **CreateAssetMenu attribute:**
   - `[CreateAssetMenu(fileName = "NewAttack", menuName = "TomatoFighters/AttackData", order = 0)]`
   - This enables right-click → Create → TomatoFighters → AttackData in the Unity Project window

### Mystica Attack Assets (4 assets)

7. **MysticaStrike1 — Magic Burst 1:**
   - `damageMultiplier: 0.6`
   - `knockbackForce: (1.5, 0)`
   - `launchForce: (0, 0)`
   - `hitboxStartFrame: 3`
   - `hitboxActiveFrames: 4`
   - `totalFrames: 16`
   - `animationSpeed: 1.0`
   - `telegraphType: Normal`
   - `causesWallBounce: false`
   - `causesLaunch: false`
   - `isOTGCapable: false`
   - `attackId: "mystica_strike_1"`
   - `attackName: "Magic Burst 1"`

8. **MysticaStrike2 — Magic Burst 2:**
   - `damageMultiplier: 0.8`
   - `knockbackForce: (2.0, 0.3)`
   - `launchForce: (0, 0)`
   - `hitboxStartFrame: 3`
   - `hitboxActiveFrames: 4`
   - `totalFrames: 18`
   - `animationSpeed: 1.0`
   - `telegraphType: Normal`
   - `causesWallBounce: false`
   - `causesLaunch: false`
   - `isOTGCapable: false`
   - `attackId: "mystica_strike_2"`
   - `attackName: "Magic Burst 2"`

9. **MysticaStrike3 — Magic Burst 3:**
   - `damageMultiplier: 1.0`
   - `knockbackForce: (3.0, 0.5)`
   - `launchForce: (0, 0)`
   - `hitboxStartFrame: 4`
   - `hitboxActiveFrames: 5`
   - `totalFrames: 22`
   - `animationSpeed: 1.0`
   - `telegraphType: Normal`
   - `causesWallBounce: false`
   - `causesLaunch: false`
   - `isOTGCapable: false`
   - `attackId: "mystica_strike_3"`
   - `attackName: "Magic Burst 3"`

10. **MysticaArcaneBolt — Arcane Bolt:**
    - `damageMultiplier: 1.4`
    - `knockbackForce: (2.0, 1.0)`
    - `launchForce: (0, 0)`
    - `hitboxStartFrame: 6`
    - `hitboxActiveFrames: 6`
    - `totalFrames: 30`
    - `animationSpeed: 1.0`
    - `telegraphType: Normal`
    - `causesWallBounce: false`
    - `causesLaunch: false`
    - `isOTGCapable: false`
    - `attackId: "mystica_arcane_bolt"`
    - `attackName: "Arcane Bolt"`

## File Plan

| File Path | Description |
|-----------|-------------|
| `Shared/Data/AttackData.cs` | ScriptableObject class defining all attack parameters |
| `ScriptableObjects/Attacks/Mystica/MysticaStrike1.asset` | Mystica's 1st combo hit — Magic Burst 1 |
| `ScriptableObjects/Attacks/Mystica/MysticaStrike2.asset` | Mystica's 2nd combo hit — Magic Burst 2 |
| `ScriptableObjects/Attacks/Mystica/MysticaStrike3.asset` | Mystica's 3rd combo hit — Magic Burst 3 |
| `ScriptableObjects/Attacks/Mystica/MysticaArcaneBolt.asset` | Mystica's heavy — Arcane Bolt |
| `Combat/Combo/ComboStep.cs` | Add `AttackData attackData` field (non-breaking) |
| `Shared/Interfaces/IAttacker.cs` | Change `CurrentAttack` from `object` to `AttackData` |

**Note:** The `.asset` files are Unity-serialized binaries created in the Unity Editor via the CreateAssetMenu. The agent should generate a C# Editor script (`Editor/CreateMysticaAttacks.cs`) that programmatically creates these assets with the correct values, since CLI agents cannot open the Unity Editor. This script runs once from `Tools > TomatoFighters > Create Mystica Attacks`.

## Design Decisions

### DD-1: ComboStep keeps damageMultiplier alongside AttackData reference (Option A)
ComboStep gains an `AttackData attackData` field but retains its existing `damageMultiplier` as a combo-specific scaling override. Final damage = `baseATK * attackData.damageMultiplier * comboStep.damageMultiplier`. The ComboStep multiplier defaults to 1.0, making it invisible unless intentionally set. This avoids a breaking change to existing ComboDefinition assets and allows the same AttackData to be reused across different combo trees with different per-step scaling.

```csharp
[Serializable]
public struct ComboStep
{
    public AttackData attackData;          // NEW: full attack data reference
    public float damageMultiplier;         // KEPT: combo-specific override (default 1.0)
    // ... existing fields unchanged
}
```

### DD-2: Mystica first instead of Brutor
Initial attack assets are created for Mystica rather than Brutor. Mystica's 3-hit projectile burst chain + Arcane Bolt heavy. Her low ATK (0.5) is offset by faster animations (16-22 total frames vs Brutor's 20-36) and her Arcane Resonance passive (+5% team damage per cast). Values are tuning estimates for Phase 5/6 balancing.

### DD-3: IAttacker.CurrentAttack typed to AttackData
The existing `object CurrentAttack` placeholder on IAttacker is updated to `AttackData CurrentAttack`, resolving the TODO comment. This is a non-breaking change since no implementors exist yet (EnemyBase is T011, still pending).

## Implementation Notes

- **Shared/Data/ location:** AttackData lives in Shared because both Combat and World pillars reference it. Combat reads it for player attacks; World reads it via IAttacker.CurrentAttack for enemy attacks. Neither pillar imports the other — they both import from Shared
- **Frame-to-time conversion:** Code consuming AttackData should convert frames to seconds using `frames / 60f` (assuming 60fps target). The frame-based definition keeps timing consistent regardless of actual framerate, and frame numbers are more intuitive for animators
- **Asset values are initial estimates:** The damage multipliers, knockback forces, and frame timing are educated guesses. They will be tuned extensively in Phase 5 (T045 — Character Combo Feel Pass) and Phase 6 (T054 — Combat Balancing). Don't over-optimize these numbers now
- **No combo branching data in AttackData:** Combo tree structure (which attack leads to which) is in ComboNode (T004), NOT in AttackData. AttackData is pure attack parameters — it doesn't know about combos. This separation means the same AttackData can be reused in different combo trees or by enemies
- **Inspector usability:** Use `[Header]` groups: "Damage", "Animation & Timing", "Telegraph", "Combo", "Effects". Use `[Tooltip]` on every field. Use `[Range]` on bounded fields: `damageMultiplier` (0.1, 5.0), `hitboxStartFrame` (0, 30), `animationSpeed` (0.1, 3.0)
- **Editor script for asset creation:** Since this project runs through CLI agents that can't open Unity Editor, include an Editor script at `Editor/CreateMysticaAttacks.cs` that uses `AssetDatabase.CreateAsset()` to programmatically create the 4 Mystica attack SO assets with all values pre-filled. Menu item: `Tools > TomatoFighters > Create Mystica Attacks`

## Acceptance Criteria

- [ ] AttackData ScriptableObject class with all required fields: damageMultiplier, knockbackForce, launchForce, animationClip, hitboxStartFrame, hitboxActiveFrames, totalFrames, animationSpeed, telegraphType, causesWallBounce, causesLaunch, isOTGCapable, isAirAttack, attackId, attackName
- [ ] `[CreateAssetMenu]` attribute with menuName "TomatoFighters/AttackData"
- [ ] TelegraphType field using the shared enum from T001 (Normal/Unstoppable)
- [ ] VFX/SFX reference fields (hitEffectPrefab, swingSound, hitSound) — nullable for Phase 1
- [ ] Inspector organized with `[Header]`, `[Tooltip]`, `[Range]` attributes
- [ ] Mystica attack set: 4 assets with correct values (Strike1, Strike2, Strike3, ArcaneBolt)
- [ ] ComboStep updated with `AttackData attackData` field (DD-1, non-breaking)
- [ ] IAttacker.CurrentAttack updated from `object` to `AttackData` (DD-3)
- [ ] Editor script to programmatically create Mystica attack assets
- [ ] All fields use documented units (frames for timing, multipliers for damage, Vector2 for forces)
- [ ] Compiles with zero warnings

## Unlocks: Cancel System Testing

T005 completion (AttackData with frame-based hitbox timing) combined with T015 (HitboxManager wiring hit detection to `OnHitConfirmed()`) unlocks end-to-end testing of the cancel system implemented in T004.

**Cancel system overview:** Players can dash-cancel or jump-cancel out of combo attacks, but only after a hit connects (hit-confirm). Without hit-confirm, cancel inputs are buffered for ~0.15s. The cancel system lives in `ComboController.TryDashCancel()` / `TryJumpCancel()` with priority resolution via `ComboInteractionConfig`.

**What T005 enables:** AttackData provides `hitboxStartFrame`, `hitboxActiveFrames`, and `totalFrames` — the frame data that HitboxManager (T015) needs to know when to activate hitboxes and report hit-confirms. Without this data, there's no automated path from "attack animation plays" to "hit confirmed."

**Tests to add after T005 + T015:**
- PlayMode integration tests: attack → hitbox activates → collides with enemy → `OnHitConfirmed()` fires → cancel window opens → dash/jump cancel executes
- Cancel buffer expiry: buffer cancel input before hit, verify it's consumed when hit-confirm arrives within window
- Per-character cancel availability: verify `canDashCancelOnHit` / `canJumpCancelOnHit` flags on ComboStep interact correctly with AttackData timing
- See `.claude/dumps/cancel-buffer-tests-dump.md` for 11 planned PlayMode test cases (unit-level, callable now without T005)

## References

- `architecture/interface-contracts.md` — AttackData field definitions, IAttacker.CurrentAttack reference
- `architecture/data-flow.md` — Step 2: ComboSystem loads AttackData from node; Step 5: damage calculation uses damageMultiplier
- `design-specs/CHARACTER-ARCHETYPES.md` — Mystica: "3-hit magical projectile burst (short range, piercing). Low damage but safe. Arcane bolt — medium range, homing, moderate damage."
- `developer/coding-standards.md` — ScriptableObjects for ALL data, [CreateAssetMenu], [Header]/[Tooltip] conventions
