# Tomato Fighters — Architecture Plan

## Vision

Build a 2D beat 'em up roguelite with 4 stat-differentiated characters, each offering 3 upgrade paths (Main + Secondary selection), layered on top of an Absolum-inspired combat system with defensive depth and elemental ritual stacking.

The architecture is split into **3 pillars** (Combat, Roguelite, World) that communicate ONLY through shared interface contracts. This enables 3 developers to work in parallel with zero coupling, coordinated through AgentPilot's context-optimized pipeline.

## Architecture

### Three-Pillar System

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          TOMATO FIGHTERS                                │
├──────────────────┬──────────────────────┬───────────────────────────────┤
│   PILLAR 1       │   PILLAR 2           │   PILLAR 3                   │
│   COMBAT         │   ROGUELITE          │   WORLD                      │
│   SANDBOX        │   SYSTEMS            │   & CONTENT                  │
├──────────────────┼──────────────────────┼───────────────────────────────┤
│ Owner: Dev 1     │ Owner: Dev 2         │ Owner: Dev 3                 │
│                  │                      │                               │
│ Character        │ Path system          │ Wave manager                 │
│ controllers (4)  │ (12 paths, tiers)    │ & level bounds               │
│                  │                      │                               │
│ Combo state      │ Stat calculator      │ Enemy AI                     │
│ machines (4)     │ (base+path+ritual)   │ (state machines)             │
│                  │                      │                               │
│ Hitbox system    │ Ritual families (8)  │ Boss AI                      │
│ (anim events)    │ & stacking           │ (phases + punish)            │
│                  │                      │                               │
│ Deflect / Clash  │ Trinket system       │ Branching path               │
│ / Dodge          │                      │ navigation                   │
│                  │                      │                               │
│ Pressure / Stun  │ Meta-progression     │ Camera system                │
│                  │ (Soul Tree)          │                               │
│                  │                      │                               │
│ Wall bounce +    │ Save / Load          │ HUD / UI                     │
│ air juggles      │                      │                               │
│                  │                      │                               │
│ Path ability     │ Hub area logic       │ Character animations         │
│ execution        │                      │ (4 base + overrides)         │
│                  │                      │                               │
│ Character        │ Path selection UI    │ Path ability VFX             │
│ passives         │ Currency manager     │ Co-op framework              │
├──────────────────┼──────────────────────┼───────────────────────────────┤
│ AP Config:       │ AP Config:           │ AP Config:                   │
│ unity-combat     │ unity-roguelite      │ unity-world                  │
├──────────────────┴──────────────────────┴───────────────────────────────┤
│                     SHARED LAYER (all 3 pillars)                        │
│  Assets/Scripts/Shared/                                                 │
│  ├── Interfaces/  (ICombatEvents, IBuffProvider, IPathProvider, etc.)   │
│  ├── Data/        (AttackData, DamagePacket, PathData, etc.)            │
│  ├── Enums/       (DamageType, PathType, CharacterType, etc.)           │
│  └── Events/      (SO-based event channels for cross-domain signals)    │
└─────────────────────────────────────────────────────────────────────────┘
```

### Character Upgrade Path Pipeline

```
Character Selected (Brutor/Slasher/Mystica/Viper)
    │
    ▼
Base Stats Loaded (from CharacterBaseStats ScriptableObject)
    │── HP, DEF, ATK, SPD, MNA, MRG, CRT, PRS
    │── Passive ability activated
    │
    ▼
Area 1 Cleared → Upgrade Shrine
    │
    ├── Choose MAIN PATH (1 of 3)
    │   └── Tier 1 unlocked: core mechanic + stat bonus
    │
    ▼
Area 2 Cleared → Upgrade Shrine
    │
    ├── Choose SECONDARY PATH (1 of remaining 2)
    │   └── Tier 1 unlocked: supplementary mechanic + stat bonus
    │
    ▼
Area Boss Defeated
    │
    ├── Main Path → Tier 2 (passive enhancement + bigger stat bonus)
    ├── Secondary Path → Tier 2 (passive enhancement)
    │
    ▼
Character Ultimate — 1 charge per run (rare to earn more)
    │── Brutor: Unbreakable | Slasher: Phantom Strike
    │── Mystica: Time Warp  | Viper: Rain of Arrows

Final Stats = Base + PathBonuses × RitualMultiplier × TrinketMultiplier × SoulTreeBonus
```

### Technology Stack

| Category | Technology | Rationale |
|----------|-----------|-----------|
| Engine | Unity 2022 LTS (2D URP) | Industry standard for 2D games, team familiarity |
| Language | C# | Unity's native language |
| Data | ScriptableObjects | Inspector-friendly, hot-reloadable, no singletons |
| Input | Unity Input System | Multi-device support, rebindable, buffer-friendly |
| Physics | Rigidbody2D | Knockback, launch, wall bounce — physics-driven |
| Animation | Animator + Overrides | Base controller per archetype, overrides per character |
| Juice | DOTween | Hitstop, screen shake, easing curves |
| Co-op | Fishnet/Mirror (TBD) | Rollback netcode for fighting-game precision |
| AI Coordination | AgentPilot | Context-optimized task execution for 3-dev team |

### Project Structure

```
tomato-fighters/
├── Assets/
│   ├── Scripts/
│   │   ├── Combat/                 ← Dev 1: combo, hitbox, defense, pressure
│   │   ├── Characters/             ← Dev 1: controllers, passives, path execution
│   │   ├── Roguelite/              ← Dev 2: rituals, trinkets, meta, save/load
│   │   ├── Paths/                  ← Dev 2: path system, stat calc, selection
│   │   ├── World/                  ← Dev 3: enemies, bosses, waves, camera, UI
│   │   └── Shared/                 ← ALL: interfaces, data, enums, events
│   ├── ScriptableObjects/
│   │   ├── Characters/ (4)         ← Base stat definitions
│   │   ├── Paths/ (12)             ← Path tier definitions
│   │   ├── Attacks/                ← Attack data per character
│   │   ├── Inspirations/ (24)      ← Character-specific unlocks
│   │   ├── Rituals/                ← 8 elemental families
│   │   ├── Trinkets/               ← Stat modifier items
│   │   ├── Enemies/                ← Enemy stat configs
│   │   └── Islands/ (4)            ← Island path definitions
│   ├── Animations/                 ← 4 base + 4 override controllers
│   ├── Prefabs/                    ← Characters, enemies, effects, pickups
│   └── Scenes/                     ← Hub, islands, bosses
├── .claude/                        ← Claude Code integration
│   ├── CLAUDE.md
│   ├── agents/ (7)
│   ├── commands/
│   └── skills/ (4)
└── config-bank/                    ← AgentPilot configs
    ├── agents/ (6)
    └── strategies/ (3)
```

### Phase Overview

| Phase | Name | Tasks | Weeks | Focus |
|-------|------|-------|-------|-------|
| **1** | Foundation | T001-T013 (13) | 1-2 | Shared contracts, basic movement, stat framework |
| **2** | Core Combat + Paths | T014-T025 (12) | 3-4 | Combo systems, path framework, enemy AI |
| **3** | Defensive Depth | T026-T034 (9) | 5-6 | Deflect/clash, rituals, boss AI |
| **4** | Advanced + Meta | T035-T044 (10) | 7-8 | Juggle/wallbounce, save/load, hub |
| **5** | Content + Co-op | T045-T052 (8) | 9-10 | All characters polished, 2 islands, co-op |
| **6** | Polish + Full Loop | T053-T060 (8) | 11-12 | Game feel, balance, 4 islands, vertical slice |

**Critical Path:** T001 (shared interfaces) → T002/T006/T011 (per-pillar foundations) → T014/T018/T022 (core systems) → T028/T032 (path + boss integration) → T060 (full playtest)
