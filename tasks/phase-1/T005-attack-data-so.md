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
| **Status** | PENDING |
| **Branch** | `pillar1/T005-attack-data-so` |

## Objective
Define the AttackData ScriptableObject — the universal data container for every attack in the game — and create Brutor's initial 4-attack chain as concrete asset instances. This SO is referenced by ComboNode, HitboxManager, EnemyAI, and the damage pipeline, making it one of the most-referenced data types in the project.

## Context
Every attack in Tomato Fighters — player strikes, enemy swings, boss slams, path abilities — is defined by an AttackData ScriptableObject. This is the data-driven core of the combat system: no attack parameters are hardcoded in scripts.

The AttackData definition is specified in `architecture/interface-contracts.md` under "Shared Data Structures". It lives in `Shared/Data/` because both Combat (reads it for player attacks) and World (reads it for enemy attacks via IAttacker) reference it.

For Phase 1, this task creates:
1. The AttackData class definition
2. Brutor's 4 attack assets: Strike 1 (Shield Jab), Strike 2 (Shield Swipe), Strike 3 (Shield Bash), Finisher (Overhead Slam)

These assets will be referenced by ComboNode instances in T004 (combo chain) and later by HitboxManager in T015.

From the character spec, Brutor's base ATK is 0.7 (low) and his strikes are described as "slow, short range, solid knockback." The finisher is "overhead shield slam, hits grounded enemies (OTG capable)."

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

### Brutor Attack Assets (4 assets)

7. **BrutorStrike1 — Shield Jab:**
   - `damageMultiplier: 1.0`
   - `knockbackForce: (3.0, 0.5)`
   - `launchForce: (0, 0)`
   - `hitboxStartFrame: 4`
   - `hitboxActiveFrames: 3`
   - `totalFrames: 20`
   - `animationSpeed: 1.0`
   - `telegraphType: Normal`
   - `causesWallBounce: false`
   - `causesLaunch: false`
   - `isOTGCapable: false`
   - `attackId: "brutor_strike_1"`
   - `attackName: "Shield Jab"`

8. **BrutorStrike2 — Shield Swipe:**
   - `damageMultiplier: 1.2`
   - `knockbackForce: (4.0, 1.0)`
   - `launchForce: (0, 0)`
   - `hitboxStartFrame: 5`
   - `hitboxActiveFrames: 4`
   - `totalFrames: 24`
   - `animationSpeed: 1.0`
   - `telegraphType: Normal`
   - `causesWallBounce: false`
   - `causesLaunch: false`
   - `isOTGCapable: false`
   - `attackId: "brutor_strike_2"`
   - `attackName: "Shield Swipe"`

9. **BrutorStrike3 — Shield Bash:**
   - `damageMultiplier: 1.5`
   - `knockbackForce: (6.0, 2.0)`
   - `launchForce: (0, 0)`
   - `hitboxStartFrame: 6`
   - `hitboxActiveFrames: 4`
   - `totalFrames: 28`
   - `animationSpeed: 0.9`
   - `telegraphType: Normal`
   - `causesWallBounce: true`
   - `causesLaunch: false`
   - `isOTGCapable: false`
   - `attackId: "brutor_strike_3"`
   - `attackName: "Shield Bash"`

10. **BrutorFinisher — Overhead Slam:**
    - `damageMultiplier: 2.0`
    - `knockbackForce: (3.0, 0)`
    - `launchForce: (0, 0)`
    - `hitboxStartFrame: 8`
    - `hitboxActiveFrames: 5`
    - `totalFrames: 36`
    - `animationSpeed: 0.8`
    - `telegraphType: Normal`
    - `causesWallBounce: false`
    - `causesLaunch: false`
    - `isOTGCapable: true`
    - `attackId: "brutor_finisher"`
    - `attackName: "Overhead Slam"`

## File Plan

| File Path | Description |
|-----------|-------------|
| `Shared/Data/AttackData.cs` | ScriptableObject class defining all attack parameters |
| `ScriptableObjects/Attacks/Brutor/BrutorStrike1.asset` | Brutor's 1st combo hit — Shield Jab |
| `ScriptableObjects/Attacks/Brutor/BrutorStrike2.asset` | Brutor's 2nd combo hit — Shield Swipe |
| `ScriptableObjects/Attacks/Brutor/BrutorStrike3.asset` | Brutor's 3rd combo hit — Shield Bash |
| `ScriptableObjects/Attacks/Brutor/BrutorFinisher.asset` | Brutor's finisher — Overhead Slam |

**Note:** The `.asset` files are Unity-serialized binaries created in the Unity Editor via the CreateAssetMenu. The agent should generate a C# Editor script (`Editor/CreateBrutorAttacks.cs`) that programmatically creates these assets with the correct values, since CLI agents cannot open the Unity Editor. This script runs once from `Tools > TomatoFighters > Create Brutor Attacks`.

## Implementation Notes

- **Shared/Data/ location:** AttackData lives in Shared because both Combat and World pillars reference it. Combat reads it for player attacks; World reads it via IAttacker.CurrentAttack for enemy attacks. Neither pillar imports the other — they both import from Shared
- **Frame-to-time conversion:** Code consuming AttackData should convert frames to seconds using `frames / 60f` (assuming 60fps target). The frame-based definition keeps timing consistent regardless of actual framerate, and frame numbers are more intuitive for animators
- **Asset values are initial estimates:** The damage multipliers, knockback forces, and frame timing are educated guesses. They will be tuned extensively in Phase 5 (T045 — Character Combo Feel Pass) and Phase 6 (T054 — Combat Balancing). Don't over-optimize these numbers now
- **No combo branching data in AttackData:** Combo tree structure (which attack leads to which) is in ComboNode (T004), NOT in AttackData. AttackData is pure attack parameters — it doesn't know about combos. This separation means the same AttackData can be reused in different combo trees or by enemies
- **Inspector usability:** Use `[Header]` groups: "Damage", "Animation & Timing", "Telegraph", "Combo", "Effects". Use `[Tooltip]` on every field. Use `[Range]` on bounded fields: `damageMultiplier` (0.1, 5.0), `hitboxStartFrame` (0, 30), `animationSpeed` (0.1, 3.0)
- **Editor script for asset creation:** Since this project runs through CLI agents that can't open Unity Editor, include an Editor script at `Editor/CreateBrutorAttacks.cs` that uses `AssetDatabase.CreateAsset()` to programmatically create the 4 Brutor attack SO assets with all values pre-filled. Menu item: `Tools > TomatoFighters > Create Brutor Attacks`

## Acceptance Criteria

- [ ] AttackData ScriptableObject class with all required fields: damageMultiplier, knockbackForce, launchForce, animationClip, hitboxStartFrame, hitboxActiveFrames, totalFrames, animationSpeed, telegraphType, causesWallBounce, causesLaunch, isOTGCapable, isAirAttack, attackId, attackName
- [ ] `[CreateAssetMenu]` attribute with menuName "TomatoFighters/AttackData"
- [ ] TelegraphType field using the shared enum from T001 (Normal/Unstoppable)
- [ ] VFX/SFX reference fields (hitEffectPrefab, swingSound, hitSound) — nullable for Phase 1
- [ ] Inspector organized with `[Header]`, `[Tooltip]`, `[Range]` attributes
- [ ] Brutor attack chain: 4 assets with correct values (Strike1, Strike2, Strike3, Finisher)
- [ ] Brutor Finisher marked as `isOTGCapable: true`
- [ ] Brutor Strike3 marked as `causesWallBounce: true`
- [ ] Editor script to programmatically create Brutor attack assets
- [ ] All fields use documented units (frames for timing, multipliers for damage, Vector2 for forces)
- [ ] Compiles with zero warnings

## References

- `architecture/interface-contracts.md` — AttackData field definitions, IAttacker.CurrentAttack reference
- `architecture/data-flow.md` — Step 2: ComboSystem loads AttackData from node; Step 5: damage calculation uses damageMultiplier
- `design-specs/CHARACTER-ARCHETYPES.md` — Brutor: "3-hit shield bash combo. Slow, short range, solid knockback. Overhead shield slam (OTG capable)"
- `developer/coding-standards.md` — ScriptableObjects for ALL data, [CreateAssetMenu], [Header]/[Tooltip] conventions
