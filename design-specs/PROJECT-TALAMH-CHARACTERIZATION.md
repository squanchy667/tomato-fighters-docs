# PROJECT TALAMH — Characterization File

**Version:** 1.0.0
**Codename:** Talamh (from Absolum's world name)
**Type:** 2D Side-Scrolling Beat 'Em Up Roguelite
**Engine:** Unity 2022 LTS (2D URP)
**Team:** 3 Developers + Agent Pilot
**Reference Game:** Absolum (Dotemu / Guard Crush, 2025)

---

## Vision Statement

A hand-drawn 2D beat 'em up where combat depth comes from **defensive mastery** (deflect, clash, punish), not button mashing. Fused with a roguelite layer where **elemental ritual families** stack into broken builds that feel different every run. Played across **hand-crafted branching maps** with choices that change the run.

The player should feel like they're getting better at TWO things simultaneously: fighting skill AND build-crafting knowledge.

---

## The Three Pillars

Everything in this project falls under one of three pillars. Each pillar is an independent domain with its own Agent Pilot config, context strategy, agents, and dev owner. Pillars communicate ONLY through the shared interface contracts in `Assets/Scripts/Shared/`.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        PROJECT TALAMH                                    │
├──────────────────┬──────────────────────┬───────────────────────────────┤
│   PILLAR 1       │   PILLAR 2           │   PILLAR 3                   │
│   COMBAT         │   ROGUELITE          │   WORLD                      │
│   SANDBOX        │   SYSTEMS            │   & CONTENT                  │
├──────────────────┼──────────────────────┼───────────────────────────────┤
│ Owner: Dev 1     │ Owner: Dev 2         │ Owner: Dev 3                 │
│                  │                      │                               │
│ Combo state      │ Ritual families      │ Wave manager                 │
│ machine          │ & stacking           │ & level bounds               │
│                  │                      │                               │
│ Hitbox system    │ Trinket system       │ Enemy AI                     │
│ (anim events)    │                      │ (state machines)             │
│                  │                      │                               │
│ Deflect / Clash  │ Inspiration system   │ Boss AI                      │
│ / Dodge          │                      │ (phases + punish windows)    │
│                  │                      │                               │
│ Pressure / Stun  │ Arcana (mana         │ Branching path               │
│ meter            │ specials)            │ navigation                   │
│                  │                      │                               │
│ Wall bounce +    │ Meta-progression     │ Quest system                 │
│ air juggles      │ (Soul Tree)          │ & world state                │
│                  │                      │                               │
│ Input buffer     │ Save / Load          │ Camera system                │
│                  │                      │                               │
│ Anti-spam        │ Hub area logic       │ HUD / UI                     │
│ (Repetitive)     │                      │                               │
│                  │ Reward selection     │ Mounts & companions          │
│ Character        │ (pick 1-of-N)        │                               │
│ controller       │                      │ Co-op framework              │
│                  │ Currency manager     │                               │
│                  │ (3 currencies)       │ Art pipeline integration     │
├──────────────────┼──────────────────────┼───────────────────────────────┤
│ AP Config:       │ AP Config:           │ AP Config:                   │
│ unity-combat     │ unity-roguelite      │ unity-world                  │
│                  │                      │                               │
│ AP Strategy:     │ AP Strategy:         │ AP Strategy:                 │
│ combat           │ roguelite            │ world                        │
├──────────────────┴──────────────────────┴───────────────────────────────┤
│                     SHARED LAYER (all 3 pillars)                        │
│  Assets/Scripts/Shared/                                                 │
│  ├── Interfaces/  (ICombatEvents, IBuffProvider, IDamageable, etc.)     │
│  ├── Data/        (AttackData, DamagePacket, etc.)                      │
│  ├── Enums/       (DamageType, RitualTrigger, TelegraphType, etc.)      │
│  └── Events/      (SO-based event channels for cross-domain signals)    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Pillar 1: COMBAT SANDBOX

### Purpose
The moment-to-moment action — how the player fights. This is the feel-layer. Every frame matters.

### Design Principles
1. **Depth through defense.** Deflect, Clash, and Punish are the primary skill expression. Players push INTO enemy attacks, not away from them.
2. **Combos emerge from cancels, not canned chains.** Dash-cancel on hit, jump-cancel, wall-bounce extensions. Strike × 3 → Finisher is the FLOOR, not the ceiling.
3. **Anti-spam enforced.** The Repetitive penalty reduces damage for spamming the same move. Forces diverse combat.
4. **All timing is configurable.** Deflect windows, i-frame durations, input buffer frames — all exposed in Inspector via ScriptableObjects. Tuning is king.

### Core Systems

| System | Key Classes | Description |
|--------|------------|-------------|
| **Character Controller** | `CharacterController2D` | 8-directional movement, jump (with gravity), forward dash, vertical dodge. Rigidbody2D physics. |
| **Input Buffer** | `InputBufferSystem` | Queue of recent inputs. Buffer window ~6 frames (0.1s). Allows pre-buffering attacks during animations. |
| **Combo State Machine** | `ComboSystem`, `ComboNode` | Tree of branching attack routes. Each node: AttackData ref, input window, cancel flags (dash/jump), launch/wallbounce flags. |
| **Hitbox Manager** | `HitboxManager` | Activates/deactivates attack colliders via Animation Events. Never in Update(). Reports hit-confirm for cancel system. |
| **Defense System** | `DefenseSystem` | Deflect (dash-timed, generous window: 0-150ms), Clash (heavy-attack-timed, tighter: 20-80ms), Dodge (i-frames: 50-300ms). Returns `DamageResponse` enum. |
| **Pressure / Stun** | `PressureSystem` | Hidden meter on enemies. Punish damage fills faster. Full meter → stunned (free juggle for ~3s) → invulnerable recovery (blink white, camera zoom). |
| **Wall Bounce** | `WallBounceHandler` | Detect wall collision mid-knockback → bounce. Unlimited per combo. Each bounce = minor damage, no extra pressure. Key for combo extensions. |
| **Air Juggle** | `JuggleSystem` | Track airborne state. Gale element extends airtime. After stun expires → invulnerable landing. OTG vs Tech Hit distinction (Arcana hits bypass Tech recovery). |
| **Anti-Spam** | `RepetitiveTracker` | Track consecutive same-move usage. After threshold → damage penalty multiplier. Resets on using different move. |
| **Attack Data** | `AttackData` (ScriptableObject) | Damage, knockback force, launch force, animation clip, hitbox timing, telegraph type (Normal/Unstoppable), combo branch links. |

### Fires Events (ICombatEvents)
```
OnStrike, OnSkill, OnArcana, OnDash, OnDeflect, OnClash, OnPunish, OnKill, 
OnFinisher, OnJump, OnDodge, OnTakeDamage
```
→ Consumed by Pillar 2 (Roguelite) for ritual triggers.

### Queries (IBuffProvider)
```
GetDamageMultiplier(type), GetSpeedMultiplier(), GetDefenseMultiplier(),
GetAdditionalOnHitEffects(), GetTriggerEffects(trigger)
```
→ Provided by Pillar 2 (Roguelite) to modify combat numbers at runtime.

---

## Pillar 2: ROGUELITE SYSTEMS

### Purpose
The meta-layer that makes every run different. Build-crafting, progression, and the "one more run" loop.

### Design Principles
1. **Broken builds are the goal.** The best runs should feel overpowered through synergy, not grind. Bramble Knives + Echo + Chain Lightning = screen-clearing madness.
2. **Elemental families as mental models.** Players think "I'm going Lightning this run" — focusing their decision-making.
3. **Characters feel incomplete at start.** Controversial but intentional. Inspirations unlock core moves mid-run. The delta between basic and fully kitted IS the roguelite dopamine.
4. **Three-currency economy.** Crystals (common, meta-tree), Imbued Fruits (rare, ritual upgrades), Primordial Seeds (rare, arcana/inspiration unlocks).

### Core Systems

| System | Key Classes | Description |
|--------|------------|-------------|
| **Ritual System** | `RitualSystem`, `RitualData` (SO) | Manages active rituals per run. Subscribes to ICombatEvents. Fires ritual effects through trigger→effect pipeline. |
| **8 Elemental Families** | `RitualFamily` enum | Fire (burn), Lightning (chain), Water (tidal waves), Thorn (bramble knives), Gale (extended juggles), Time (echoes), Cosmic (ultimates), Necro (lifesteal). |
| **Ritual Categories** | `RitualCategory` enum | Core (main mechanic), General (global buff), Enhancement (amplify existing), Twin (cross-family combo — requires both families present). |
| **Stacking Calculator** | `RitualStackCalculator` | Plain C# class. Level scaling: L1=base, L2=1.5×, L3=2.0×. Same-mechanic stacking: 2 rituals=1.5×, 3=2.0×. Ritual Power from trinkets is multiplicative with itself (1.1 × 1.1 = 1.21×). |
| **Trinket System** | `TrinketSystem`, `TrinketData` (SO) | Stat modifiers found during runs. Boost Ritual Power (Moxes), grant bonuses on perfect dodge, increase rare drop chance, etc. |
| **Inspiration System** | `InspirationSystem`, `InspirationData` (SO) | Character-specific move unlocks found during runs. Dropped by minibosses. Galandra's Dive Kick, Karl's Explosive Punch, etc. |
| **Arcana System** | `ArcanaSystem`, `ArcanaData` (SO) | Mana-consuming special abilities. Selected before run. Second Arcana unlocked mid-run after Island 1. Glowing Arcana = bonus XP. |
| **Meta-Progression** | `MetaProgression`, `SoulTree` | Persistent between runs. Soul Tree: health+, damage+, self-revives, rare chance. Currencies: Crystals, Imbued Fruits, Primordial Seeds. |
| **Reward Selection** | `RewardSelectorUI` | Post-area: pick 1 of 2 rituals (or 3 with Soul Tree upgrade), or gold, or crystals. |
| **Currency Manager** | `CurrencyManager` | Central authority for all currency modifications. Fires events on change. Crystals carry between runs. |
| **Save / Load** | `SaveSystem` | JSON serialization to Application.persistentDataPath. Stores MetaProgressionData, unlocked rituals, character unlocks, quest state. |
| **Hub Area** | `HubManager` | The Hearth — base between runs. NPCs: Vikhana (unlock Twin Rituals), Ederig (unlock Inspirations). Soul Tree, shops, character/arcana selection. |

### Subscribes To (ICombatEvents)
Listens for all combat triggers to fire ritual effects.

### Provides (IBuffProvider)
Returns damage/speed/defense multipliers and on-hit effects based on active ritual + trinket build.

### Subscribes To (IRunProgressionEvents)
Listens for OnAreaCleared → show reward selection. OnBossDefeated → special rewards. OnRunEnded → persist currencies.

---

## Pillar 3: WORLD & CONTENT

### Purpose
Everything the player navigates through and fights against. The run structure, enemies, bosses, environments, and presentation layer.

### Design Principles
1. **Fixed maps, branching paths.** NOT procedurally generated. Hand-crafted encounters with route choices at forks. Cheaper to build, better encounter design, still varied.
2. **Enemy telegraphs are the fairness contract.** Normal attacks = can deflect/clash. Red flash = Unstoppable = must dodge. This is non-negotiable.
3. **Bosses teach through failure.** Each boss has readable phases, clear punish windows, and escalating patterns. Stun system rewards sustained aggression.
4. **4 islands, play 3 per run.** Fork choices determine which 3 you visit. Different enemies, cultures, bosses per island.

### Core Systems

| System | Key Classes | Description |
|--------|------------|-------------|
| **Wave Manager** | `WaveManager`, `Wave`, `LevelBound` | Spawn enemies in waves. Camera stops at LevelBound until wave cleared. Configurable enemy composition per wave. |
| **Enemy Base** | `EnemyBase` (implements IDamageable + IAttacker) | Health, pressure meter, knockback handling, invulnerability after stun recovery. All enemies inherit from this. |
| **Enemy AI** | `EnemyAI`, `EnemyStateBase` | State machine: Idle → Patrol → Chase → Attack → HitReact → Death. Configurable aggression, attack frequency, telegraph duration. |
| **Boss AI** | `BossAI`, `BossPhase` | Extends EnemyAI with Phase system. Phases transition by HP% thresholds. Each phase: own attack pattern list (SO), new moves, faster tempo. Punish windows after big attacks. Camera zoom on stun. |
| **Branching Paths** | `PathNavigator`, `PathNodeData` (SO), `IslandData` (SO) | Map UI showing route options. Nodes: Combat, Shop, Challenge, Boss, Inspiration, Story. Fork choices at island transitions. |
| **Island Manager** | `IslandManager` | 4 islands total, player traverses 3. Each island: unique enemy set, sub-bosses, environment, culture, final boss. Randomized enemy types within fixed map layouts. |
| **Quest System** | `QuestSystem`, `QuestData` (SO) | Side quests unlock routes, NPCs, rewards. Tracked on map. World state changes based on quest completion. |
| **Camera** | `CameraController2D` | Side-scroll follow with configurable leading. Level bound stops. Zoom-in on stun events (listens to PressureSystem event). Co-op framing for 2 players. |
| **HUD / UI** | `HUDManager`, `HealthBarUI`, `ComboCounterUI`, `ManaBarUI` | Health bars: world-space canvas on enemies. HUD: screen-space overlay for player stats, combo counter, mana. |
| **Mounts & Companions** | `MountSystem`, `CompanionAI` | Rideable mounts found in certain areas. AI mercenaries hireable. Both have combat capabilities. |
| **Co-op** | `CoopManager` | 2-player: local couch + online (rollback netcode TBD). Players don't share gold. Separate build paths. |
| **Art Pipeline** | Animator Override Controllers | Base Animator Controller with shared state machine. Per-character overrides. Animation Events for hitbox/VFX/SFX timing. |

### Fires Events (IRunProgressionEvents)
```
OnAreaCleared, OnBossDefeated, OnIslandCompleted, OnShopEntered, OnRunStarted, OnRunEnded
```
→ Consumed by Pillar 2 for reward selection and currency persistence.

### Implements (IDamageable + IAttacker)
All enemies implement the combat interfaces. Combat system hits enemies through these contracts — never touches enemy AI internals.

---

## Interface Contracts

The shared boundary between all three pillars. Lives in `Assets/Scripts/Shared/Interfaces/`. Changes require ALL 3 devs to review and approve.

### ICombatEvents (Pillar 1 fires → Pillar 2 subscribes)
```csharp
public interface ICombatEvents
{
    event Action<StrikeEventData> OnStrike;
    event Action<DashEventData> OnDash;
    event Action<DeflectEventData> OnDeflect;
    event Action<ClashEventData> OnClash;
    event Action<PunishEventData> OnPunish;
    event Action<KillEventData> OnKill;
    event Action<FinisherEventData> OnFinisher;
    event Action<ArcanaEventData> OnArcana;
    event Action<JumpEventData> OnJump;
    event Action<DodgeEventData> OnDodge;
    event Action<TakeDamageEventData> OnTakeDamage;
}
```

### IBuffProvider (Pillar 2 provides → Pillar 1 queries)
```csharp
public interface IBuffProvider
{
    float GetDamageMultiplier(DamageType type);
    float GetSpeedMultiplier();
    float GetDefenseMultiplier();
    List<OnHitEffect> GetAdditionalOnHitEffects();
    List<OnTriggerEffect> GetTriggerEffects(RitualTrigger trigger);
    bool IsRepetitivePenaltyOverridden();
}
```

### IDamageable (Pillar 1 defines → Pillar 3 implements on enemies)
```csharp
public interface IDamageable
{
    void TakeDamage(DamagePacket damage);
    float CurrentHealth { get; }
    float MaxHealth { get; }
    void AddPressure(float amount);
    bool IsStunned { get; }
    void ApplyKnockback(Vector2 force);
    void ApplyLaunch(Vector2 force);
    bool IsInvulnerable { get; }
}
```

### IAttacker (Pillar 1 defines → Pillar 3 implements on enemies)
```csharp
public interface IAttacker
{
    AttackData CurrentAttack { get; }
    bool IsCurrentAttackUnstoppable { get; }
    TelegraphType CurrentTelegraphType { get; }
    float PunishWindowDuration { get; }
    bool IsInPunishableState { get; }
}
```

### IRunProgressionEvents (Pillar 3 fires → Pillar 2 subscribes)
```csharp
public interface IRunProgressionEvents
{
    event Action<AreaClearedData> OnAreaCleared;
    event Action<BossDefeatedData> OnBossDefeated;
    event Action<IslandCompletedData> OnIslandCompleted;
    event Action<ShopEnteredData> OnShopEntered;
    event Action OnRunStarted;
    event Action<RunEndData> OnRunEnded;
}
```

### Shared Enums
```csharp
public enum DamageType { Physical, Fire, Lightning, Water, Thorn, Gale, Time, Cosmic, Necro }
public enum RitualTrigger { OnStrike, OnSkill, OnDash, OnDeflect, OnClash, OnFinisher, OnKill, OnArcana, OnJump, OnDodge, OnTakeDamage }
public enum TelegraphType { Normal, Unstoppable }
public enum RitualFamily { Fire, Lightning, Water, Thorn, Gale, Time, Cosmic, Necro }
public enum RitualCategory { Core, General, Enhancement, Twin }
public enum BossPhase { Phase1, Phase2, Phase3, Enraged, Stunned }
```

---

## Agent Pilot Configuration

### Directory Structure
```
ProjectTalamh/
├── .claude/
│   ├── CLAUDE.md                     # Project conventions (Agent Pilot reads this always)
│   ├── agents/
│   │   ├── combat-engineer.md        # Pillar 1 agent definition
│   │   ├── roguelite-architect.md    # Pillar 2 agent definition
│   │   ├── world-builder.md          # Pillar 3 agent definition
│   │   ├── integration-wirer.md      # Cross-pillar integration agent
│   │   ├── combat-feel-tuner.md      # Game feel specialist
│   │   ├── balance-analyst.md        # Ritual/enemy balance analysis
│   │   └── test-writer.md            # Unit test specialist
│   ├── commands/
│   │   ├── do-task.md                # Single task through pipeline
│   │   ├── plan-sprint.md            # Decompose sprint goal into tasks
│   │   ├── execute-phase.md          # Run all tasks in a phase
│   │   ├── build-pillar.md           # Full orchestration for one pillar
│   │   ├── scan-project.md           # Index Unity project for context
│   │   ├── integration-test.md       # Cross-pillar validation
│   │   └── capture-learnings.md      # Feed back into config bank
│   └── skills/
│       ├── token-budgeting.md        # Token budget awareness
│       ├── context-handoff.md        # Phase-to-phase summaries
│       ├── unity-conventions.md      # Unity-specific patterns
│       └── interface-guard.md        # Prevent cross-pillar violations
├── config-bank/
│   ├── agents/
│   │   ├── unity-combat.json         # Config: combat domain scoring
│   │   ├── unity-roguelite.json      # Config: roguelite domain scoring
│   │   ├── unity-world.json          # Config: world domain scoring
│   │   ├── unity-integration.json    # Config: cross-domain tasks
│   │   └── unity-combat-feel.json    # Config: game feel tuning
│   └── strategies/
│       ├── combat.json               # File prioritization: Combat/ + Shared/
│       ├── roguelite.json            # File prioritization: Roguelite/ + Shared/
│       └── world.json                # File prioritization: World/ + Shared/
└── Assets/Scripts/                   # Unity source
```

### Config Bank Agent Configs (JSON)

Each config targets one pillar. Agent Pilot's weighted scoring (domain 0.4 + stack 0.35 + type 0.25) automatically routes tasks to the right specialist.

| Config ID | Domains | Stack | Task Types | Strategy |
|-----------|---------|-------|------------|----------|
| `unity-combat` | combat, physics, animation, input | unity, csharp, rigidbody2d, animator, inputsystem | implementation, refactor, debug | `combat` |
| `unity-roguelite` | roguelite, progression, data, ui | unity, csharp, scriptableobject, json, eventsystem | implementation, refactor, debug | `roguelite` |
| `unity-world` | ai, world, enemies, camera, ui | unity, csharp, animator, statemachine, canvas | implementation, refactor, debug | `world` |
| `unity-integration` | combat, roguelite, world | unity, csharp | implementation, refactor | `fullstack` |
| `unity-combat-feel` | combat, animation, physics | unity, csharp, dotween | refactor | `combat` |

### Context Strategies (JSON)

Each strategy defines which project files an agent sees:

| Strategy | Prioritizes | Excludes |
|----------|------------|----------|
| `combat` | `**/Combat/**`, `**/Shared/**`, `**/ScriptableObjects/Attacks/**` | `**/Roguelite/**`, `**/World/**`, `**/*.unity`, `**/*.meta` |
| `roguelite` | `**/Roguelite/**`, `**/Shared/**`, `**/ScriptableObjects/Rituals/**`, `**/ScriptableObjects/Trinkets/**` | `**/Combat/**`, `**/World/**`, `**/*.unity`, `**/*.meta` |
| `world` | `**/World/**`, `**/Shared/**`, `**/ScriptableObjects/Enemies/**`, `**/ScriptableObjects/Islands/**` | `**/Combat/**`, `**/Roguelite/**`, `**/*.unity`, `**/*.meta` |

### Skills (Always-On Context)

| Skill | Purpose |
|-------|---------|
| `token-budgeting` | Enforce ≤15K tokens per agent. Compression: FULL → SUMMARY → REFERENCE → SKIP |
| `context-handoff` | Generate ~1K token phase summaries for downstream phases |
| `unity-conventions` | Unity-specific: Animation Events for timing, SO for data, no singletons, Rigidbody2D for physics |
| `interface-guard` | **CRITICAL.** Flags any code that imports across pillar boundaries without going through Shared/Interfaces/. Prevents coupling. |

---

## Parallel Execution Plan

### Sync Protocol

| Cadence | Event | Who | Agent Pilot Command |
|---------|-------|-----|-------------------|
| **Daily** | Each dev runs 1-3 tasks through pipeline | Individual | `agent-pilot "<task>" --context . --budget 10000` |
| **2× per week** | Integration build: merge all branches, compile, quick playtest | All 3 | `/integration-test` |
| **Weekly** | Playtest session: play together, log feel issues | All 3 | Manual |
| **Bi-weekly** | Sprint review: demo, retrospect, update configs | All 3 | `/capture-learnings` |
| **As needed** | Interface change: propose → review → approve → update | All 3 | Manual review + `/scan-project` |

### Phase Plan (First 6 Sprints)

```
Sprint 1 (Week 1-2): FOUNDATION
───────────────────────────────
  Pillar 1: CharacterController2D, InputBuffer, basic Strike chain, AttackData SO
  Pillar 2: RitualData SO, enums/shared types, CurrencyManager, RitualStackCalculator
  Pillar 3: WaveManager, EnemyBase (IDamageable), CameraController, basic scene
  Shared:   Interface contracts, shared enums, DamagePacket struct
  ▸ Integration target: Character moves, hits a dummy enemy, camera follows

Sprint 2 (Week 3-4): CORE MECHANICS
────────────────────────────────────
  Pillar 1: ComboSystem (branching nodes), HitboxManager, dash-cancel on hit
  Pillar 2: RitualSystem (trigger pipeline), TrinketSystem, Fire + Lightning families
  Pillar 3: BasicEnemyAI (state machine), enemy attack patterns, telegraph visuals
  ▸ Integration target: Fight through one wave, collect a ritual, see it modify damage

Sprint 3 (Week 5-6): DEFENSIVE DEPTH + BUILD CRAFTING
──────────────────────────────────────────────────────
  Pillar 1: DefenseSystem (Deflect/Clash/Dodge), PressureSystem, Punish multiplier
  Pillar 2: Reward selection UI, Inspiration system, Arcana system
  Pillar 3: BossAI framework (phases, punish windows), branching path navigation
  ▸ Integration target: Full loop — fight area → pick ritual → fork in path → fight boss

Sprint 4 (Week 7-8): ADVANCED COMBAT + META-PROGRESSION
────────────────────────────────────────────────────────
  Pillar 1: WallBounce, AirJuggle, OTG vs TechHit, RepetitiveTracker
  Pillar 2: MetaProgression (Soul Tree), SaveLoad, Hub area logic, remaining ritual families
  Pillar 3: 2nd enemy type, mini-bosses, quest system, HUD (health/mana/combo)
  ▸ Integration target: Complete run from hub → 2 areas → boss → back to hub with progress

Sprint 5 (Week 9-10): CONTENT + CO-OP
──────────────────────────────────────
  Pillar 1: 2nd playable character, character-specific combos, Animator Overrides
  Pillar 2: Twin Rituals, ritual balance pass, all 8 families complete
  Pillar 3: 2nd island, local co-op, mounts/companions, more enemy variety
  ▸ Integration target: 2 characters, 2 islands, co-op on couch

Sprint 6 (Week 11-12): POLISH + FULL LOOP
──────────────────────────────────────────
  Pillar 1: Game feel pass (hitstop, screen shake, juice), combat balancing
  Pillar 2: Build balance pass, currency economy tuning, difficulty scaling
  Pillar 3: All 4 islands, online co-op (if feasible), VFX pass, sound integration
  ▸ Integration target: Full playable vertical slice — hub → 3 islands → final boss
```

### Parallel Task Execution Per Sprint

Within each sprint, Agent Pilot's `getParallelGroups()` identifies tasks with no dependencies that can run simultaneously. Example for Sprint 2:

```
Parallel Group 1 (no dependencies between these):
  ├── [Dev 1] COMBAT-005: ComboSystem branching node tree
  ├── [Dev 2] RITUAL-007: RitualSystem trigger→effect pipeline  
  └── [Dev 3] WORLD-007: BasicEnemyAI state machine

Parallel Group 2 (depends on Group 1):
  ├── [Dev 1] COMBAT-006: HitboxManager with Animation Events
  ├── [Dev 2] RITUAL-008: Fire family rituals (Burn, Blazing Dash)
  └── [Dev 3] WORLD-008: Enemy attack patterns + telegraph visuals

Parallel Group 3 (depends on Group 2):
  ├── [Dev 1] COMBAT-007: Dash-cancel on hit-confirm integration
  ├── [Dev 2] RITUAL-009: Lightning family (Chain Lightning, Lightning Strike)
  └── [Dev 3] WORLD-009: Wire EnemyBase → combat hitbox collision

Integration Group (all 3 devs):
  └── INTEGRATION-002: Ritual buff → combat damage → enemy IDamageable full loop
```

---

## Quality Gates

Inherited from Agent Pilot's pipeline. Applied per-sprint:

| Gate | Threshold | What It Checks |
|------|-----------|----------------|
| **Task Quality** | ≥ 70/100 per task | Completeness + correctness + conventions + token efficiency |
| **Phase Pass Rate** | ≥ 60% tasks pass | Minimum tasks succeeding before proceeding |
| **Phase Avg Quality** | ≥ 50 average | Minimum quality floor |
| **Integration Build** | Compiles + runs | All 3 pillars merged, no errors, basic gameplay loop works |
| **Playtest Gate** | Subjective | Does it FEEL right? Combat devs must sign off on feel. |

---

## Conventions

### Code
- **C# naming:** PascalCase classes/methods, camelCase fields, UPPER_SNAKE constants
- **No singletons.** Use `[SerializeField]` injection or SO-based event channels
- **ScriptableObjects for ALL data.** Attacks, rituals, enemies, trinkets, paths, quests
- **Animation Events for timing.** Hitbox activation, VFX spawning, SFX triggering
- **Rigidbody2D for physics.** Knockback, launch, gravity. Never manipulate transform.position for physics effects
- **Comments:** WHY, not WHAT. Every public API has summary XML doc

### Agent Pilot
- **All tasks go through the 8-step pipeline.** No cowboy coding.
- **Context strategy enforced.** Agents never see code from other pillars.
- **Interface-guard skill** catches cross-pillar imports at validation step.
- **Task log maintained.** Every agent task → logged with quality score, iterations, lessons.
- **Capture learnings** after every sprint → feed config bank flywheel.

### Git
```
main                              ← Always buildable
├── pillar1/combat-*              ← Dev 1 feature branches
├── pillar2/roguelite-*           ← Dev 2 feature branches
├── pillar3/world-*               ← Dev 3 feature branches
└── integration/sprint-N          ← Merge target before main
```

---

## Appendix: Characters (Content Targets)

| Character | Archetype | Weapon | Unlock |
|-----------|-----------|--------|--------|
| **Galandra** | All-rounder Swordfighter | Colossal Sword + Necromancy | Start |
| **Karl** | Tanky Brawler | Blunderbuss + Fists | Start |
| **Cider** | Agile Skirmisher | Clockwork Prosthetics | Unlock mid-game |
| **Brome** | Ranged Wizard | Frog Magic | Unlock mid-game |

*These are Absolum's archetypes for reference. Our game will have different characters with similar mechanical roles.*

## Appendix: Ritual Families (Content Targets)

| Family | Core Mechanic | Example Triggers |
|--------|--------------|-----------------|
| **Fire** | Burn DoT | Blazing Dash, Flame Strike |
| **Lightning** | Chain damage | Lightning Strike, Shock Wave |
| **Water** | Tidal waves | Wave Dash, Wave Landing |
| **Thorn** | Spawn projectiles | Bramble Knives on finisher, on dodge |
| **Gale** | Extended airtime | Juggle extension, lift on strike |
| **Time** | Echo/clone attacks | Echo on Arcana, slow field |
| **Cosmic** | Ultimate generation | AoE burst, infinite ultimate |
| **Necro** | Lifesteal / summons | Heal on kill, skeleton minions |
