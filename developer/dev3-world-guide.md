# Dev 3 Guide — World & Content

**Owner:** Dev 3
**Pillar:** World & Content + Visual Layer
**Branch prefix:** `pillar3/world-*`
**AgentPilot config:** `unity-world`
**Context strategy:** `world` (sees: `World/`, `Shared/`, `Animations/`)

---

## Your Domain

You own everything the player sees, navigates through, and fights against:

| System | Key Files | What It Does |
|--------|----------|-------------|
| Wave Manager | `World/WaveManager.cs` | Enemy spawning, level bounds |
| Enemy Base | `World/EnemyBase.cs` | IDamageable + IAttacker base class |
| Enemy AI | `World/EnemyAI.cs`, states | 6-state machine per enemy |
| Boss AI | `World/BossAI.cs`, `BossPhase.cs` | Phase system, punish windows |
| Branching Paths | `World/PathNavigator.cs` | Route choices, island navigation |
| Camera | `World/CameraController2D.cs` | Follow, bounds, zoom, co-op framing |
| HUD/UI | `World/UI/HUDManager.cs` | Health, mana, combo counter, path indicator |
| Quest System | `World/QuestSystem.cs` | Side quests, world state |
| Character Animation | Animator Controllers + Overrides | 4 base + 4 overrides |
| Path Ability VFX | Effect prefabs | VFX for all 36 path abilities |
| Co-op | `World/CoopManager.cs` | Local + online 2-player |
| Mounts/Companions | `World/MountSystem.cs`, `CompanionAI.cs` | Rideable mounts, AI allies |

## You Do NOT Touch

- `Combat/` — Dev 1's domain (enemies interact through IDamageable/IAttacker interfaces)
- `Roguelite/` — Dev 2's domain
- `Paths/` — Dev 2's domain

## How You Communicate with Other Pillars

| Direction | Interface | What Happens |
|-----------|-----------|-------------|
| Dev 1 → You | `IDamageable` | Dev 1's combat calls TakeDamage on your enemies through the interface |
| Dev 1 → You | `IAttacker` | Dev 1 reads your enemies' attack data to determine deflect/clash response |
| You → Dev 2 | `IRunProgressionEvents` | You **fire** area/boss/island events → Dev 2 handles rewards, progression |
| Dev 2 → You | `IPathProvider` | You display path info in HUD (query path state for UI display) |

## Enemy Design Rules

| Rule | Description |
|------|-------------|
| Normal attacks | Wind-up animation, can be deflected/clashed by player |
| Unstoppable (red) | Red flash overlay, MUST dodge, cannot deflect/clash |
| Boss phases | Transition at HP% thresholds, each phase = new attack pattern SO |
| Punish windows | After boss big attacks, configurable duration, player does bonus damage |
| Stun integration | When PressureSystem stuns, enemy enters stun state → free juggle → invulnerable recovery |
| Telegraph duration | Configurable per attack, longer = more fair, shorter = harder |

## Character Visual Identity

| Character | Animation Style | Effect Theme | Color |
|-----------|----------------|-------------|-------|
| Brutor | Heavy, slow recovery, weighted | Earth, shield glow, shockwaves | Brown/Orange |
| Slasher | Snappy, fast recovery, sharp | Speed lines, afterimages, slash trails | Red/Crimson |
| Mystica | Floaty, smooth, magical | Sparkles, aura glow, summon portals | Purple/Blue |
| Viper | Sharp, precise, mechanical | Projectile trails, target reticles, traps | Green/Yellow |

## Your Task Sequence

```
Phase 1: T010, T011, T012, T013 (waves, enemy base, camera, test scene)
Phase 2: T022 → T023, T024, T025 (enemy AI, attacks, animators, HUD)
Phase 3: T032, T033, T034 (boss AI, branching paths, T1 VFX)
Phase 4: T042, T043, T044 (2nd enemy, quests, T2/T3 VFX)
Phase 5: T050, T051, T052 (2nd island, co-op, mounts)
Phase 6: T057, T058, T059 (islands 3+4, VFX/sound, online co-op)
```

## Running a Task

```bash
# Single task through existing pipeline:
/do-task "T011: EnemyBase implementing IDamageable + IAttacker. Health, pressure meter, knockback via Rigidbody2D, invulnerable recovery with blink white. Reads stats from EnemyData ScriptableObject."

# Or for a full phase:
/execute-phase 1
```
