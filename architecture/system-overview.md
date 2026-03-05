# System Overview

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      TOMATO FIGHTERS                             │
│                                                                  │
│  ┌──────────┐    ┌──────────────┐    ┌─────────────────┐       │
│  │ COMBAT   │    │  ROGUELITE   │    │  WORLD          │       │
│  │ (Dev 1)  │    │  (Dev 2)     │    │  (Dev 3)        │       │
│  │          │    │              │    │                  │       │
│  │ Combo    │◄──►│ Rituals      │    │ Enemy AI        │       │
│  │ Hitbox   │    │ Paths        │    │ Boss AI         │       │
│  │ Defense  │    │ Stats        │    │ Waves           │       │
│  │ Pressure │    │ Trinkets     │    │ Camera          │       │
│  │ Juggle   │    │ Meta-prog    │    │ HUD             │       │
│  │ Passives │    │ Save/Load    │    │ Animation       │       │
│  │ PathExec │    │ Hub          │    │ VFX             │       │
│  └────┬─────┘    └──────┬───────┘    └────┬────────────┘       │
│       │                 │                  │                    │
│       └─────────┬───────┴──────────┬───────┘                   │
│                 │                  │                            │
│       ┌─────────▼──────────────────▼─────────┐                 │
│       │          SHARED LAYER                 │                 │
│       │                                       │                 │
│       │  ICombatEvents   (P1 → P2)            │                 │
│       │  IBuffProvider   (P2 → P1)            │                 │
│       │  IPathProvider   (P2 → P1,P3)         │                 │
│       │  IDamageable     (P1 → P3)            │                 │
│       │  IAttacker       (P3 → P1)            │                 │
│       │  IRunProgression (P3 → P2)            │                 │
│       │                                       │                 │
│       │  Enums, Data Structs, SO Events       │                 │
│       └───────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────────┘
```

## Module Architecture

```
Assets/Scripts/
├── Combat/              (Dev 1) — 11 files, ~2,500 LOC target
│   ├── CharacterController2D.cs
│   ├── InputBufferSystem.cs
│   ├── ComboSystem.cs
│   ├── ComboNode.cs
│   ├── HitboxManager.cs
│   ├── DefenseSystem.cs
│   ├── PressureSystem.cs
│   ├── WallBounceHandler.cs
│   ├── JuggleSystem.cs
│   └── ArcanaSystem.cs
├── Characters/          (Dev 1) — 16+ files, ~2,000 LOC target
│   ├── PassiveAbilitySystem.cs
│   ├── PathAbilityExecutor.cs
│   ├── Passives/        (4 passives)
│   └── Abilities/       (12 paths × 3 tiers = 36 abilities)
├── Roguelite/           (Dev 2) — 12 files, ~2,500 LOC target
│   ├── RitualSystem.cs
│   ├── RitualStackCalculator.cs
│   ├── TrinketSystem.cs
│   ├── InspirationSystem.cs
│   ├── MetaProgression.cs
│   ├── SoulTree.cs
│   ├── CurrencyManager.cs
│   ├── SaveSystem.cs
│   ├── HubManager.cs
│   ├── PathSelectionUI.cs
│   ├── RewardSelectorUI.cs
│   └── DifficultyScaling.cs
├── Paths/               (Dev 2) — 4 files, ~800 LOC target
│   ├── PathSystem.cs
│   ├── CharacterStatCalculator.cs
│   ├── FinalStats.cs
│   └── StatModifierInput.cs
├── World/               (Dev 3) — 15+ files, ~3,000 LOC target
│   ├── WaveManager.cs
│   ├── EnemyBase.cs
│   ├── EnemyAI.cs
│   ├── EnemyStateBase.cs
│   ├── States/          (6 state files)
│   ├── BossAI.cs
│   ├── BossPhase.cs
│   ├── PathNavigator.cs
│   ├── CameraController2D.cs
│   ├── CoopManager.cs
│   ├── MountSystem.cs
│   ├── CompanionAI.cs
│   ├── QuestSystem.cs
│   └── UI/              (HUD files)
└── Shared/              (ALL) — 15+ files, ~600 LOC target
    ├── Interfaces/      (6 interface files)
    ├── Data/            (CharacterBaseStats ✓, PathData ✓, PathTierBonuses ✓, AttackData, DamagePacket, etc.)
    ├── Data/            (AttackData, DamagePacket, PathData, etc.)
    ├── Enums/           (all shared enums)
    └── Events/          (SO-based event channels)
```

## Estimated Totals

| Module | Files | LOC Target | Owner |
|--------|-------|-----------|-------|
| Combat | 11 | ~2,500 | Dev 1 |
| Characters | 16+ | ~2,000 | Dev 1 |
| Roguelite | 12 | ~2,500 | Dev 2 |
| Paths | 3 | ~800 | Dev 2 |
| World | 15+ | ~3,000 | Dev 3 |
| Shared | 15+ | ~600 | ALL |
| **Total** | **~72+** | **~11,400** | |

## Key Design Decisions

- **Interface-only coupling** — Pillars never import from each other's namespaces directly
- **ScriptableObjects for ALL data** — No hardcoded values, everything tunable in Inspector
- **No singletons** — Dependency injection via `[SerializeField]` or SO-based event channels
- **Animation Events for timing** — Hitbox activation, VFX, SFX never in Update()
- **Rigidbody2D for physics** — Knockback, launch, bounce through physics, not transform manipulation
- **Plain C# for testable logic** — Stat calculators, stacking math, state machines as pure C# classes
- **Main + Secondary path** — Creates 6 builds per character (24 total) without 12 full implementations
- **Script-driven asset generation** — Prefabs, attack data, combo definitions, animation controllers, and hitbox setups are all generated by editor scripts (`Assets/Editor/`), not hand-built in the Inspector. When a design decision changes, we update the script and re-run it. This means the scripts define what the game's assets look like — the `.asset` and `.prefab` files are just their output. See [Unity Editor Scripts](../developer/unity-editor-scripts.md) for the full catalog and workflow recipes
