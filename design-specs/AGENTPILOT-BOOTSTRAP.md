# TOMATO FIGHTERS — AgentPilot Bootstrap & 3-Developer Coordination Guide

**Version:** 1.0.0
**Purpose:** Bootstrap the project from design docs into a working AgentPilot-coordinated development pipeline
**Prerequisite Files:**
- `PROJECT-TALAMH-CHARACTERIZATION.md` — Game systems & pillars
- `CHARACTER-ARCHETYPES.md` — 4 characters, paths, upgrade system
- `absolum-reverse-engineering-unity-guide.md` — Reference game mechanics

---

## Step 1: Establish the Centralized Knowledge Base

Before any code is written, all 3 developers must share a single source of truth. This is the **Project Knowledge Base (PKB)** — a docs repo that AgentPilot agents pull context from.

### 1.1 Repository Structure

```
TomatoFighters/
├── tomato-fighters/                    ← Code repo (Unity project)
│   ├── Assets/
│   │   ├── Scripts/
│   │   │   ├── Combat/                 ← Dev 1 domain
│   │   │   ├── Characters/             ← Dev 1 domain (movesets, controllers)
│   │   │   ├── Roguelite/              ← Dev 2 domain
│   │   │   ├── Paths/                  ← Dev 2 domain (upgrade path system)
│   │   │   ├── World/                  ← Dev 3 domain
│   │   │   └── Shared/                 ← ALL 3 devs (interfaces, data, enums, events)
│   │   │       ├── Interfaces/
│   │   │       ├── Data/
│   │   │       ├── Enums/
│   │   │       └── Events/
│   │   ├── ScriptableObjects/
│   │   │   ├── Characters/             ← Base stats per character
│   │   │   ├── Paths/                  ← 12 path definitions (4 chars × 3 paths)
│   │   │   ├── Attacks/                ← Attack data per character
│   │   │   ├── Inspirations/           ← 24 inspirations (4 chars × 6 each)
│   │   │   ├── Rituals/               ← Elemental ritual families
│   │   │   ├── Trinkets/
│   │   │   ├── Enemies/
│   │   │   └── Islands/
│   │   ├── Animations/
│   │   ├── Prefabs/
│   │   └── Scenes/
│   ├── .claude/
│   │   ├── CLAUDE.md                   ← Project conventions
│   │   ├── agents/                     ← 7 agent definitions
│   │   ├── commands/                   ← Slash commands
│   │   └── skills/                     ← Always-on context
│   └── config-bank/
│       ├── agents/                     ← 6 AgentPilot configs (JSON)
│       └── strategies/                 ← 3 context strategies (JSON)
│
├── tomato-fighters-docs/               ← Docs repo (this repo)
│   ├── README.md
│   ├── SUMMARY.md
│   ├── PLAN.md
│   ├── TASK_BOARD.md
│   ├── design-specs/
│   │   ├── PROJECT-TALAMH-CHARACTERIZATION.md
│   │   ├── CHARACTER-ARCHETYPES.md
│   │   ├── absolum-reverse-engineering-unity-guide.md
│   │   └── AGENTPILOT-BOOTSTRAP.md     ← This file
│   ├── architecture/
│   │   ├── system-overview.md
│   │   ├── interface-contracts.md      ← The boundary between pillars
│   │   └── data-flow.md
│   ├── developer/
│   │   ├── setup-guide.md
│   │   ├── coding-standards.md
│   │   └── dev1-guide.md / dev2-guide.md / dev3-guide.md
│   └── tasks/
│       ├── phase-1/
│       ├── phase-2/
│       └── ...
```

### 1.2 What Goes Where

| Document | Lives In | Who Maintains | Purpose |
|----------|----------|---------------|---------|
| Interface Contracts | `architecture/interface-contracts.md` | ALL 3 devs (co-authored) | The API boundary between pillars |
| Character Archetypes | `design-specs/CHARACTER-ARCHETYPES.md` | Product owner | Character definitions, stats, paths |
| Game Systems | `design-specs/PROJECT-TALAMH-CHARACTERIZATION.md` | Product owner | Combat, roguelite, world pillars |
| Task Board | `TASK_BOARD.md` | Updated by whoever completes tasks | Phase tables with status tracking |
| Coding Standards | `developer/coding-standards.md` | ALL 3 devs | Unity conventions, naming, patterns |

---

## Step 2: Define Interface Contracts (Day 1 — All 3 Devs Together)

This is the most important step. The interface contracts are what allow parallel development. All 3 devs sit together and agree on these contracts before writing ANY gameplay code.

### 2.1 Contracts to Define

The existing Talamh contracts still apply, but need extensions for the character/path system:

```csharp
// === COMBAT EVENTS (Dev 1 fires → Dev 2 subscribes) ===
public interface ICombatEvents
{
    event Action<StrikeEventData> OnStrike;
    event Action<SkillEventData> OnSkill;
    event Action<ArcanaEventData> OnArcana;
    event Action<DashEventData> OnDash;
    event Action<DeflectEventData> OnDeflect;
    event Action<ClashEventData> OnClash;
    event Action<PunishEventData> OnPunish;
    event Action<KillEventData> OnKill;
    event Action<FinisherEventData> OnFinisher;
    event Action<JumpEventData> OnJump;
    event Action<DodgeEventData> OnDodge;
    event Action<TakeDamageEventData> OnTakeDamage;
    // NEW: Path ability events
    event Action<PathAbilityEventData> OnPathAbilityUsed;
}

// === BUFF PROVIDER (Dev 2 provides → Dev 1 queries) ===
public interface IBuffProvider
{
    float GetDamageMultiplier(DamageType type);
    float GetSpeedMultiplier();
    float GetDefenseMultiplier();
    List<OnHitEffect> GetAdditionalOnHitEffects();
    List<OnTriggerEffect> GetTriggerEffects(RitualTrigger trigger);
    bool IsRepetitivePenaltyOverridden();
    // NEW: Path-specific queries
    float GetPathDamageMultiplier();
    float GetPathDefenseMultiplier();
    float GetPathSpeedMultiplier();
    List<PathAbility> GetActivePathAbilities();
}

// === PATH PROVIDER (Dev 2 provides → Dev 1 + Dev 3 query) ===
public interface IPathProvider
{
    CharacterType Character { get; }
    PathData MainPath { get; }
    PathData SecondaryPath { get; }
    int MainPathTier { get; }       // 1-3
    int SecondaryPathTier { get; }   // 1-2
    bool HasPath(PathType type);
    float GetPathStatBonus(StatType stat);
    bool IsPathAbilityUnlocked(string abilityId);
}

// === DAMAGEABLE (Dev 1 defines → Dev 3 implements on enemies) ===
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

// === ATTACKER (Dev 1 defines → Dev 3 implements on enemies) ===
public interface IAttacker
{
    AttackData CurrentAttack { get; }
    bool IsCurrentAttackUnstoppable { get; }
    TelegraphType CurrentTelegraphType { get; }
    float PunishWindowDuration { get; }
    bool IsInPunishableState { get; }
}

// === RUN PROGRESSION (Dev 3 fires → Dev 2 subscribes) ===
public interface IRunProgressionEvents
{
    event Action<AreaClearedData> OnAreaCleared;
    event Action<BossDefeatedData> OnBossDefeated;
    event Action<IslandCompletedData> OnIslandCompleted;
    event Action<ShopEnteredData> OnShopEntered;
    event Action OnRunStarted;
    event Action<RunEndData> OnRunEnded;
    // NEW: Path selection events
    event Action<PathSelectedData> OnMainPathSelected;
    event Action<PathSelectedData> OnSecondaryPathSelected;
    event Action<PathTierUpData> OnPathTierUp;
}
```

### 2.2 Shared Enums (Extended)

```csharp
public enum CharacterType { Brutor, Slasher, Mystica, Viper }

public enum PathType
{
    // Brutor
    Warden, Bulwark, Guardian,
    // Slasher
    Executioner, Reaper, Shadow,
    // Mystica
    Sage, Enchanter, Conjurer,
    // Viper
    Marksman, Trapper, Arcanist
}

public enum StatType
{
    Health, Defense, Attack, RangedAttack, Speed,
    Mana, ManaRegen, CritChance, PressureRate
}

// Existing enums unchanged
public enum DamageType { Physical, Fire, Lightning, Water, Thorn, Gale, Time, Cosmic, Necro }
public enum RitualTrigger { OnStrike, OnSkill, OnDash, OnDeflect, OnClash, OnFinisher, OnKill, OnArcana, OnJump, OnDodge, OnTakeDamage }
public enum TelegraphType { Normal, Unstoppable }
public enum RitualFamily { Fire, Lightning, Water, Thorn, Gale, Time, Cosmic, Necro }
public enum RitualCategory { Core, General, Enhancement, Twin }
```

---

## Step 3: Developer Role Assignment

### Dev 1 — Attack Mechanics + Character Combat

**Pillar:** Combat Sandbox + Character Movesets
**Branch prefix:** `pillar1/combat-*`
**Owns:**

| System | Key Scripts | Description |
|--------|-----------|-------------|
| Character Controller | `CharacterController2D.cs` | 8-dir movement, jump, dash per character |
| Input Buffer | `InputBufferSystem.cs` | 6-frame buffer, pre-buffering during animations |
| Combo State Machine | `ComboSystem.cs`, `ComboNode.cs` | Character-specific combo trees |
| Hitbox Manager | `HitboxManager.cs` | Animation Event-driven hitbox activation |
| Defense System | `DefenseSystem.cs` | Deflect/Clash/Dodge per character (different windows) |
| Pressure/Stun | `PressureSystem.cs` | Pressure meter, stun state |
| Wall Bounce | `WallBounceHandler.cs` | Wall collision → bounce physics |
| Air Juggle | `JuggleSystem.cs` | Airborne tracking, OTG vs Tech Hit |
| Anti-Spam | `RepetitiveTracker.cs` | Damage penalty for move spam |
| **Path Abilities (Combat)** | `PathAbilityExecutor.cs` | Execute active abilities from paths (Provoke, Phase Dash, Harpoon, etc.) |
| **Character Passives** | `PassiveAbilitySystem.cs` | Thick Skin, Bloodlust, Arcane Resonance, Distance Bonus |

**Context Strategy Files:** `Combat/`, `Characters/`, `Shared/Interfaces/`, `ScriptableObjects/Attacks/`

### Dev 2 — Buff/Path System + Roguelite Layer

**Pillar:** Roguelite Systems + Character Progression
**Branch prefix:** `pillar2/roguelite-*`
**Owns:**

| System | Key Scripts | Description |
|--------|-----------|-------------|
| Path System | `PathSystem.cs`, `PathData.cs` (SO) | Main/Secondary path selection, tier progression |
| Stat Calculator | `CharacterStatCalculator.cs` | Base stats + path bonuses + ritual + trinkets = final stats |
| Ritual System | `RitualSystem.cs`, `RitualData.cs` (SO) | 8 elemental families, trigger→effect pipeline |
| Stacking Calculator | `RitualStackCalculator.cs` | Multiplicative stacking math |
| Trinket System | `TrinketSystem.cs`, `TrinketData.cs` (SO) | Stat modifiers during runs |
| Inspiration System | `InspirationSystem.cs`, `InspirationData.cs` (SO) | Character-specific move unlocks |
| Meta-Progression | `MetaProgression.cs`, `SoulTree.cs` | Persistent upgrades between runs |
| Currency Manager | `CurrencyManager.cs` | Crystals, Imbued Fruits, Primordial Seeds |
| Save/Load | `SaveSystem.cs` | JSON persistence |
| Hub Area | `HubManager.cs` | Between-run base, NPC interactions, character selection |
| Reward Selection | `RewardSelectorUI.cs` | Post-area ritual/gold/crystal picks |
| **Path Selection UI** | `PathSelectionUI.cs` | Upgrade shrine: choose Main/Secondary path |
| **Character Data** | `CharacterBaseStats.cs` (SO) | 4 character stat definitions |

**Context Strategy Files:** `Roguelite/`, `Paths/`, `Shared/Interfaces/`, `ScriptableObjects/Rituals/`, `ScriptableObjects/Paths/`, `ScriptableObjects/Characters/`

### Dev 3 — Animation/Movement/Characters + World

**Pillar:** World & Content + Visual Layer
**Branch prefix:** `pillar3/world-*`
**Owns:**

| System | Key Scripts | Description |
|--------|-----------|-------------|
| Wave Manager | `WaveManager.cs` | Enemy spawning, level bounds |
| Enemy Base | `EnemyBase.cs` | IDamageable + IAttacker implementation |
| Enemy AI | `EnemyAI.cs`, `EnemyStateBase.cs` | State machine: Idle→Patrol→Chase→Attack→HitReact→Death |
| Boss AI | `BossAI.cs`, `BossPhase.cs` | Phase system, punish windows, Unstoppable attacks |
| Branching Paths | `PathNavigator.cs`, `PathNodeData.cs` (SO) | Route choices, island navigation |
| Camera | `CameraController2D.cs` | Follow, level bounds, zoom-on-stun, co-op framing |
| HUD/UI | `HUDManager.cs`, `HealthBarUI.cs`, `ComboCounterUI.cs`, `ManaBarUI.cs` | Health/mana bars, combo counter, path indicator |
| Quest System | `QuestSystem.cs` | Side quests, world state changes |
| **Character Animations** | Animator Controllers + Overrides | Base controller + per-character animation overrides |
| **Character Visual Setup** | Prefab construction | Sprite setup, hitbox colliders, animation events |
| **Path Ability VFX** | VFX prefabs | Visual effects for all 12 path abilities |
| Co-op | `CoopManager.cs` | Local + online 2-player |

**Context Strategy Files:** `World/`, `Shared/Interfaces/`, `ScriptableObjects/Enemies/`, `ScriptableObjects/Islands/`, `Animations/`

---

## Step 4: AgentPilot Config Bank Setup

### 4.1 Agent Configs (JSON)

Create these 6 configs in `config-bank/agents/`:

**unity-combat.json** — For Dev 1 combat tasks
```json
{
  "id": "unity-combat",
  "name": "Tomato Fighters Combat Engineer",
  "version": "1.0.0",
  "domains": ["combat", "physics", "animation", "input", "characters"],
  "stack": ["unity", "csharp", "rigidbody2d", "animator", "inputsystem"],
  "taskTypes": ["implementation", "refactor", "debug"],
  "systemPromptTemplate": "You are a Unity C# combat systems specialist for Tomato Fighters, a 2D beat em up roguelite.\n\n4 Characters with different combat feel:\n- Brutor (Tank): slow, armored dash, shield-based, super armor\n- Slasher (Melee DPS): fast 4-hit chains, crit-on-deflect, bloodlust stacking\n- Mystica (Mage): projectile strikes, blink teleport, mana-on-deflect\n- Viper (Range): backflip dash, projectile reflect, distance bonus\n\nEach has 3 upgrade paths with active abilities executed through the combat system.\n\nRules:\n- Match interface contracts in Shared/Interfaces/ exactly\n- ScriptableObjects for ALL attack/ability data\n- Animation Events for hitbox timing\n- Input buffer: 6 frames (0.1s at 60fps)\n- Rigidbody2D for physics, never transform.position\n- All timing values configurable via Inspector\n- No singletons\n\nTask: {{task}}\nConstraints: {{constraints}}\nContext: {{context}}",
  "contextStrategy": "combat",
  "outputFormat": "C# source files for Unity. MonoBehaviours for components, plain C# for logic, ScriptableObjects for data.",
  "qualityHistory": []
}
```

**unity-roguelite.json** — For Dev 2 progression tasks
```json
{
  "id": "unity-roguelite",
  "name": "Tomato Fighters Roguelite Architect",
  "version": "1.0.0",
  "domains": ["roguelite", "progression", "data", "characters", "paths"],
  "stack": ["unity", "csharp", "scriptableobject", "json", "eventsystem"],
  "taskTypes": ["implementation", "refactor", "debug"],
  "systemPromptTemplate": "You are a Unity C# roguelite systems specialist for Tomato Fighters.\n\nKey system: 4 characters each with 3 upgrade paths. Players pick 1 Main (T1-T3) + 1 Secondary (T1-T2). Paths give stat bonuses + unlock abilities.\n\nStat framework: HP, DEF, ATK, SPD, MNA, MRG, CRT, PRS\nFinal stat = Base + PathBonus + RitualMultiplier + TrinketFlat + SoulTreePermanent\n\nRitual stacking: multiplicative (1.1 * 1.1 = 1.21, NOT 1.2)\nPath + Ritual interaction: independent systems multiplied together\n\nRules:\n- Subscribe to ICombatEvents, never reference combat MonoBehaviours\n- All data as ScriptableObjects\n- Twin Rituals require both families present\n- Save/Load via JSON to Application.persistentDataPath\n- Currency via central CurrencyManager with events\n- No singletons\n\nTask: {{task}}\nConstraints: {{constraints}}\nContext: {{context}}",
  "contextStrategy": "roguelite",
  "outputFormat": "C# source files. ScriptableObject definitions, event channels, plain C# calculators for testability.",
  "qualityHistory": []
}
```

**unity-world.json** — For Dev 3 world/content tasks
```json
{
  "id": "unity-world",
  "name": "Tomato Fighters World Builder",
  "version": "1.0.0",
  "domains": ["ai", "world", "enemies", "camera", "ui", "animation"],
  "stack": ["unity", "csharp", "animator", "statemachine", "canvas", "cinemachine"],
  "taskTypes": ["implementation", "refactor", "debug"],
  "systemPromptTemplate": "You are a Unity C# world systems specialist for Tomato Fighters.\n\nYou handle: enemy AI, wave management, boss phases, branching paths, camera, HUD/UI, character animations/VFX, and co-op.\n\n4 characters need distinct visual setups:\n- Brutor: Heavy, slow animations, shield effects\n- Slasher: Fast, multi-hit animations, afterimage effects\n- Mystica: Magical, floating, summon spawning, aura VFX\n- Viper: Ranged, projectile trails, backflip, trap placement\n\nRules:\n- All enemies implement IDamageable + IAttacker\n- TelegraphType.Unstoppable for unblockable attacks\n- Boss phases by HP% thresholds\n- Wave manager: level bounds stop camera until cleared\n- Fire IRunProgressionEvents on completion\n- Animator Override Controllers for character variants\n- Animation Events for hitbox/VFX/SFX timing\n\nTask: {{task}}\nConstraints: {{constraints}}\nContext: {{context}}",
  "contextStrategy": "world",
  "outputFormat": "C# source files, Animator setup instructions, prefab setup instructions.",
  "qualityHistory": []
}
```

**unity-path-system.json** — Cross-domain path system tasks
```json
{
  "id": "unity-path-system",
  "name": "Path System Specialist",
  "version": "1.0.0",
  "domains": ["characters", "paths", "combat", "roguelite"],
  "stack": ["unity", "csharp", "scriptableobject"],
  "taskTypes": ["implementation", "refactor"],
  "systemPromptTemplate": "You specialize in the character upgrade path system for Tomato Fighters.\n\nSystem overview:\n- 4 characters × 3 paths each = 12 paths total\n- Players select 1 Main (progresses to T3) + 1 Secondary (progresses to T2)\n- Paths provide: stat bonuses per tier + active abilities + passive modifiers\n- Path stats combine with base stats, rituals, trinkets multiplicatively\n\nPath abilities are defined in PathData ScriptableObjects but EXECUTED by the combat system through IPathProvider queries.\n\nRules:\n- PathData as ScriptableObjects with tier definitions\n- Stat calculation: Base + Sum(PathBonuses) then × RitualMultiplier × TrinketMultiplier\n- Path abilities communicate through ICombatEvents + IPathProvider\n- Never reference cross-domain code directly\n\nTask: {{task}}\nConstraints: {{constraints}}\nContext: {{context}}",
  "contextStrategy": "fullstack",
  "outputFormat": "C# source files. Data definitions, stat calculators, path progression logic.",
  "qualityHistory": []
}
```

**unity-integration.json** — Cross-pillar wiring
```json
{
  "id": "unity-integration",
  "name": "Cross-Domain Integrator",
  "version": "1.0.0",
  "domains": ["combat", "roguelite", "world", "characters"],
  "stack": ["unity", "csharp"],
  "taskTypes": ["implementation", "refactor"],
  "systemPromptTemplate": "You integrate systems across domain boundaries in Tomato Fighters.\n\nYou work ONLY through interfaces in Shared/Interfaces/.\nNever add direct references between Combat/, Roguelite/, World/ folders.\nYour job: wire events, inject dependencies, ensure contracts work at runtime.\n\nKey integration points:\n- Combat fires ICombatEvents → Roguelite subscribes for ritual triggers\n- Roguelite provides IBuffProvider → Combat queries for damage multipliers\n- Roguelite provides IPathProvider → Combat queries for path abilities\n- World fires IRunProgressionEvents → Roguelite subscribes for rewards/progression\n- World implements IDamageable+IAttacker → Combat hits enemies through interfaces\n\nTask: {{task}}\nConstraints: {{constraints}}\nContext: {{context}}",
  "contextStrategy": "fullstack",
  "outputFormat": "Integration wiring code, dependency injection setup.",
  "qualityHistory": []
}
```

**unity-combat-feel.json** — Game feel tuning
```json
{
  "id": "unity-combat-feel",
  "name": "Combat Feel Tuner",
  "version": "1.0.0",
  "domains": ["combat", "animation", "physics"],
  "stack": ["unity", "csharp", "dotween"],
  "taskTypes": ["refactor"],
  "systemPromptTemplate": "You are a game feel specialist for Tomato Fighters.\n\nYou tune: timing values, easing curves, screen shake, hitstop, juice.\nYou think in frames (60fps). 2-3 frames hitstop on heavy hits.\n\n4 characters need DIFFERENT feel:\n- Brutor: Heavy, impactful, slow recovery, earth-shaking effects\n- Slasher: Snappy, fast recovery, tight cancel windows, speed lines\n- Mystica: Floaty, magical particles, smooth transitions, sparkle effects\n- Viper: Sharp, precise, clean projectile trails, satisfying hit feedback\n\nTask: {{task}}\nConstraints: {{constraints}}\nContext: {{context}}",
  "contextStrategy": "combat",
  "outputFormat": "Modified C# files with timing values, juice effects, feel adjustments.",
  "qualityHistory": []
}
```

### 4.2 Context Strategies (JSON)

Create these 3 strategies in `config-bank/strategies/`:

```json
// combat.json
{
  "id": "combat",
  "name": "Combat Strategy",
  "domains": ["combat", "physics", "animation", "input", "characters"],
  "priorityPatterns": [
    "**/Combat/**",
    "**/Characters/**",
    "**/Shared/Interfaces/**",
    "**/Shared/Data/**",
    "**/Shared/Enums/**",
    "**/ScriptableObjects/Attacks/**",
    "**/ScriptableObjects/Characters/**"
  ],
  "excludePatterns": [
    "**/Roguelite/**",
    "**/World/**",
    "**/Paths/**",
    "**/*.unity",
    "**/*.meta",
    "**/*.asset",
    "**/Animations/**",
    "**/Prefabs/**"
  ],
  "defaultRelevance": 0.2
}
```

```json
// roguelite.json
{
  "id": "roguelite",
  "name": "Roguelite Strategy",
  "domains": ["roguelite", "progression", "data", "paths"],
  "priorityPatterns": [
    "**/Roguelite/**",
    "**/Paths/**",
    "**/Shared/Interfaces/**",
    "**/Shared/Data/**",
    "**/Shared/Enums/**",
    "**/ScriptableObjects/Rituals/**",
    "**/ScriptableObjects/Trinkets/**",
    "**/ScriptableObjects/Paths/**",
    "**/ScriptableObjects/Characters/**",
    "**/ScriptableObjects/Inspirations/**"
  ],
  "excludePatterns": [
    "**/Combat/**",
    "**/World/**",
    "**/Characters/**",
    "**/*.unity",
    "**/*.meta",
    "**/Animations/**",
    "**/Prefabs/**"
  ],
  "defaultRelevance": 0.2
}
```

```json
// world.json
{
  "id": "world",
  "name": "World Strategy",
  "domains": ["ai", "world", "enemies", "camera", "ui", "animation"],
  "priorityPatterns": [
    "**/World/**",
    "**/Shared/Interfaces/**",
    "**/Shared/Data/**",
    "**/Shared/Enums/**",
    "**/ScriptableObjects/Enemies/**",
    "**/ScriptableObjects/Islands/**",
    "**/Animations/**"
  ],
  "excludePatterns": [
    "**/Combat/**",
    "**/Roguelite/**",
    "**/Paths/**",
    "**/*.unity",
    "**/*.meta"
  ],
  "defaultRelevance": 0.2
}
```

---

## Step 5: Phase Plan — From Zero to Playable

### Phase 1: FOUNDATION (Week 1-2) — All 3 Devs Start Simultaneously

**Integration Target:** One character moves, hits a dummy enemy, camera follows.

| Task ID | Dev | Task | Dependencies | Priority |
|---------|-----|------|-------------|----------|
| T001 | ALL | Shared interfaces, enums, data structures | None | **Critical** |
| T002 | 1 | CharacterController2D (Brutor first — simplest moveset) | T001 | High |
| T003 | 1 | InputBufferSystem (6-frame buffer) | T001 | High |
| T004 | 1 | Basic Strike chain (3 hits + finisher) | T002, T003 | High |
| T005 | 1 | AttackData ScriptableObject definition | T001 | High |
| T006 | 2 | CharacterBaseStats ScriptableObject (4 characters) | T001 | High |
| T007 | 2 | CharacterStatCalculator (base stats + path bonuses) | T006 | High |
| T008 | 2 | PathData ScriptableObject (12 paths defined) | T001, T006 | High |
| T009 | 2 | CurrencyManager (3 currencies + events) | T001 | Medium |
| T010 | 3 | WaveManager (spawn enemies, level bounds) | T001 | High |
| T011 | 3 | EnemyBase (IDamageable + IAttacker) | T001 | High |
| T012 | 3 | CameraController2D (follow, bounds, zoom) | T001 | High |
| T013 | 3 | Basic test scene setup | T002, T010, T011, T012 | Medium |

**Parallel Groups:**
```
Group 1: T001 (ALL — sync session)
Group 2: T002, T003, T005 (Dev 1) | T006, T009 (Dev 2) | T010, T011, T012 (Dev 3)
Group 3: T004 (Dev 1) | T007, T008 (Dev 2) | T013 (Dev 3)
Integration: Merge, compile, basic playtest
```

### Phase 2: CORE COMBAT + PATH FRAMEWORK (Week 3-4)

**Integration Target:** All 4 characters playable with basic combos, path selection UI works, fight one wave.

| Task ID | Dev | Task | Dependencies | Priority |
|---------|-----|------|-------------|----------|
| T014 | 1 | ComboSystem with branching nodes (per character) | T004, T005 | High |
| T015 | 1 | HitboxManager (Animation Event-driven) | T014 | High |
| T016 | 1 | DefenseSystem (Deflect/Clash/Dodge) | T014 | High |
| T017 | 1 | Character passives (Thick Skin, Bloodlust, etc.) | T007, T014 | Medium |
| T018 | 2 | PathSystem (selection logic, tier progression) | T008 | High |
| T019 | 2 | PathSelectionUI (upgrade shrine) | T018 | High |
| T020 | 2 | RitualData ScriptableObject + family definitions | T001 | Medium |
| T021 | 2 | RitualSystem (trigger→effect pipeline) | T020 | Medium |
| T022 | 3 | BasicEnemyAI (state machine: 6 states) | T011 | High |
| T023 | 3 | Enemy attack patterns + telegraph visuals | T022, T005 | High |
| T024 | 3 | Character Animator Controllers (4 base + overrides) | T014 | High |
| T025 | 3 | HUD: health bars, mana bar, combo counter | T007 | Medium |

### Phase 3: DEFENSIVE DEPTH + BUILD CRAFTING (Week 5-6)

**Integration Target:** Full loop — fight area, pick ritual, select path at shrine, fight boss.

| Task ID | Dev | Task | Dependencies |
|---------|-----|------|-------------|
| T026 | 1 | PressureSystem + Stun state | T016 |
| T027 | 1 | WallBounce + AirJuggle systems | T015 |
| T028 | 1 | Path ability execution (all T1 abilities) | T017, T018 |
| T029 | 2 | RitualStackCalculator (multiplicative math) | T021 |
| T030 | 2 | TrinketSystem | T007 |
| T031 | 2 | RewardSelectorUI (post-area picks) | T021 |
| T032 | 3 | BossAI framework (phases, punish windows) | T022, T026 |
| T033 | 3 | Branching path navigation | T010 |
| T034 | 3 | Path ability VFX (all T1 abilities) | T028 |

### Phase 4: ADVANCED COMBAT + META-PROGRESSION (Week 7-8)

**Integration Target:** Complete run from hub → 2 areas → boss → hub with path progress saved.

| Task ID | Dev | Task | Dependencies |
|---------|-----|------|-------------|
| T035 | 1 | RepetitiveTracker (anti-spam) | T014 |
| T036 | 1 | OTG vs TechHit distinction | T027 |
| T037 | 1 | Path T2 + T3 ability execution | T028 |
| T038 | 2 | MetaProgression + SoulTree | T009 |
| T039 | 2 | SaveSystem (JSON persistence) | T038 |
| T040 | 2 | HubManager (between-run base) | T038, T039 |
| T041 | 2 | InspirationSystem (24 inspirations) | T018 |
| T042 | 3 | 2nd enemy type + mini-bosses | T022, T032 |
| T043 | 3 | Quest system | T033 |
| T044 | 3 | Path T2 + T3 ability VFX | T037 |

### Phase 5: CONTENT + CO-OP (Week 9-10)

**Integration Target:** 2 characters fully polished, 2 islands, co-op on couch.

| Task ID | Dev | Task | Dependencies |
|---------|-----|------|-------------|
| T045 | 1 | Character-specific combo feel pass (all 4) | T037 |
| T046 | 1 | Arcana system (mana specials per character) | T037 |
| T047 | 2 | Twin Rituals (cross-family combos) | T029 |
| T048 | 2 | All 8 ritual families complete | T029 |
| T049 | 2 | Ritual balance pass (synergy testing) | T048 |
| T050 | 3 | 2nd island (new enemies, boss, environment) | T042 |
| T051 | 3 | Local co-op | T012 |
| T052 | 3 | Mounts & companions | T022 |

### Phase 6: POLISH + FULL LOOP (Week 11-12)

**Integration Target:** Full playable vertical slice — hub → 3 islands → final boss.

| Task ID | Dev | Task | Dependencies |
|---------|-----|------|-------------|
| T053 | 1 | Game feel pass (hitstop, screen shake, juice) | T045 |
| T054 | 1 | Combat balancing (per character) | T053 |
| T055 | 2 | Build balance pass + economy tuning | T049 |
| T056 | 2 | Difficulty scaling | T055 |
| T057 | 3 | Islands 3 + 4 | T050 |
| T058 | 3 | VFX + sound integration pass | T044 |
| T059 | 3 | Online co-op (if feasible) | T051 |
| T060 | ALL | Integration testing + playtest | ALL above |

---

## Step 6: Sync Protocol

### Daily (Each Dev Independently)

```bash
# Dev runs 1-3 tasks through AgentPilot pipeline
agent-pilot "Implement CharacterController2D for Brutor with armored dash" \
  --context ./tomato-fighters --budget 10000
```

### 2x Per Week (Integration Build)

All 3 devs merge to `integration/sprint-N`, compile, quick playtest.

```bash
# Run integration validation
agent-pilot "Integration test: verify ICombatEvents → RitualSystem → IBuffProvider loop" \
  --context ./tomato-fighters --budget 8000
```

### Weekly (Playtest Session)

All 3 play together. Log feel issues. Combat dev must sign off on feel.

### Bi-Weekly (Sprint Review)

Demo, retrospect, update configs.

```bash
agent-pilot capture-learnings
```

### As Needed (Interface Changes)

**Process:**
1. Proposing dev creates a PR with interface change
2. ALL 3 devs review and approve
3. Merge → each dev updates their code to match
4. Re-run `agent-pilot scan-repo ./tomato-fighters`

---

## Step 7: Git Strategy

```
main                                ← Always buildable
├── pillar1/combat-*                ← Dev 1 feature branches
│   ├── pillar1/combat-controller
│   ├── pillar1/combat-combo-system
│   └── pillar1/combat-defense
├── pillar2/roguelite-*             ← Dev 2 feature branches
│   ├── pillar2/roguelite-path-system
│   ├── pillar2/roguelite-rituals
│   └── pillar2/roguelite-meta
├── pillar3/world-*                 ← Dev 3 feature branches
│   ├── pillar3/world-enemy-ai
│   ├── pillar3/world-boss-ai
│   └── pillar3/world-camera
└── integration/sprint-N            ← Merge target before main
```

**Branch Rules:**
- Devs ONLY touch files in their domain + `Shared/` (with review)
- Integration branches merge all 3 pillars for testing
- Only integration branches merge to `main`
- Commit format: `[Phase X] TXXX: Brief description`

---

## Step 8: Quality Gates

| Gate | Threshold | Check |
|------|-----------|-------|
| Task Quality | >= 70/100 | Completeness + correctness + conventions |
| Phase Pass Rate | >= 60% | Minimum tasks succeeding |
| Phase Avg Quality | >= 50 | Quality floor |
| Integration Build | Compiles + runs | All 3 pillars merged, no errors |
| Playtest Gate | Subjective | Does combat FEEL right? Character differentiation clear? |
| **Path Balance Gate** | All 6 builds per character viable | No path combination is strictly dominated |

---

## Step 9: Getting Started Checklist

### Day 1 (All 3 Devs Together)
- [ ] Create Unity project (2022 LTS, 2D URP)
- [ ] Set up git repo with branch strategy
- [ ] Set up docs repo with PKB structure
- [ ] Write interface contracts together (Step 2 above)
- [ ] Review CHARACTER-ARCHETYPES.md together — agree on stat values
- [ ] Create `Shared/` folder with all interfaces, enums, data structures (T001)

### Day 2 (Split to Parallel Work)
- [ ] Install AgentPilot: `npm install -g agent-pilot`
- [ ] Create `config-bank/` with 6 agent configs + 3 strategies (Step 4 above)
- [ ] Run `agent-pilot scan-repo ./tomato-fighters` to build initial index
- [ ] Each dev picks their first Phase 1 tasks
- [ ] Each dev runs their first AgentPilot task and validates output

### Week 1 (Foundation Sprint)
- [ ] Dev 1: T002-T005 (controller, input, combo, attack data)
- [ ] Dev 2: T006-T009 (stats, calculator, paths, currency)
- [ ] Dev 3: T010-T013 (waves, enemy base, camera, test scene)
- [ ] Friday: Integration build — one character moves and hits dummy enemy

### Week 2 (Foundation Completion)
- [ ] Fix integration issues from Week 1
- [ ] Playtest: does Brutor FEEL like a tank? Is the base loop satisfying?
- [ ] Update PKB with lessons learned
- [ ] `agent-pilot capture-learnings` to update config bank

---

## Appendix: AgentPilot Commands Quick Reference

```
DAILY COMMANDS
──────────────
agent-pilot "<task>" --context . --budget 10000    # Execute single task
agent-pilot execute-phase N                        # Run full phase
agent-pilot scan-repo ./tomato-fighters            # Re-index project
agent-pilot capture-learnings                      # Feed config bank

CUSTOM CONFIGS
──────────────
unity-combat.json        → Dev 1 (combo, hitbox, defense, passives)
unity-roguelite.json     → Dev 2 (rituals, paths, meta, save)
unity-world.json         → Dev 3 (enemies, bosses, camera, animation)
unity-path-system.json   → Cross-domain path tasks
unity-integration.json   → Cross-pillar wiring
unity-combat-feel.json   → Game feel tuning

TOKEN BUDGET
────────────
Default: 10K per task (7K agent + 2K test + 1K docs)
Hard ceiling: 15K per agent
Compression: FULL → SUMMARY → REFERENCE → SKIP

GOLDEN RULE
An agent that knows less, produces better output.
```
