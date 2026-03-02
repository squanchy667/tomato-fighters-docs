# Agent Pilot × Project Talamh
## Operational Guide: Building an Absolum-Style Game with 3 Developers

*Based on [squanchy667/agentpilot](https://github.com/squanchy667/agentpilot) — Context-optimized AI task execution engine*

---

## Part 1: Why Agent Pilot for Game Dev

Agent Pilot's core thesis maps perfectly to a multi-dev game project:

> A 10K-token agent with exactly the right context will outperform a 100K-token agent drowning in irrelevant context — every single time.

In a Unity beat 'em up with combat systems, roguelite progression, enemy AI, and world management, the codebase grows fast. An agent building ritual stacking math does NOT need to see combo state machine code, enemy behavior trees, or UI scripts. Agent Pilot's **config bank + context strategies + progressive compression** ensures each agent only sees what it needs.

### What Agent Pilot Actually Does

```
agent-pilot "Implement the Deflect system with i-frame timing" --context ./ProjectTalamh --budget 10000
```

This single command triggers the 8-step pipeline:
1. **Analyze** → detects domains, complexity, task type
2. **Build Agent** → scores 15 configs, selects best match, assembles compressed context
3. **Execute** → runs in clean context window (no history, no irrelevant files)
4. **Capture** → collects output files
5. **Test** → independent validation (completeness, correctness, conventions)
6. **Document** → generates audit trail (`task-spec.md`, `agent-config.md`, `summary.md`)
7. **Report** → quality score 0-100

The key: **the agent never exceeds 15K tokens** and gets only files matching its context strategy.

---

## Part 2: Project Bootstrap

### 2.1 Install Agent Pilot

```bash
# Global install
npm install -g agent-pilot

# Verify
agent-pilot --help
```

**Requires:** Node.js 20+, Claude Code CLI

### 2.2 Initialize the Unity Project for Agent Pilot

```
ProjectTalamh/
├── Assets/                          # Unity project root
│   ├── Scripts/
│   │   ├── Combat/                  # Dev 1's domain
│   │   ├── Roguelite/               # Dev 2's domain
│   │   ├── World/                   # Dev 3's domain
│   │   └── Shared/                  # Interfaces, data structures, enums
│   ├── ScriptableObjects/
│   ├── Animations/
│   └── Prefabs/
├── config-bank/                     # ← Agent Pilot custom configs
│   ├── agents/
│   │   ├── unity-combat.json        # Combat specialist agent
│   │   ├── unity-roguelite.json     # Roguelite systems agent
│   │   ├── unity-world.json         # World/enemy/AI agent
│   │   ├── unity-reviewer.json      # Code review agent
│   │   └── unity-test-writer.json   # Test writing agent
│   └── strategies/
│       ├── combat.json              # File prioritization for combat tasks
│       ├── roguelite.json           # File prioritization for roguelite tasks
│       └── world.json               # File prioritization for world tasks
├── .claude/
│   ├── commands/                    # Slash commands for Claude Code
│   └── agents/                      # Agent definitions
├── Docs/
│   ├── design-specs/
│   │   ├── combat-design.md
│   │   ├── roguelite-design.md
│   │   └── world-design.md
│   └── absolum-reverse-engineering.md
└── package.json                     # For Agent Pilot deps
```

### 2.3 Scan the Project (Once It Has Code)

After the initial scaffolding exists, run the scanner to build a context index:

```bash
agent-pilot scan-repo ./ProjectTalamh
```

This uses Agent Pilot's **scanner module** to:
- **`indexFiles`** → Walk all directories, classify 30+ languages, detect domains
- **`mapDependencies`** → Build import/dependency graph between files
- **`detectPatterns`** → Detect frameworks, naming conventions, test runners
- **`buildContextIndex`** → Persist unified index to `.agent-pilot-index.json`

From then on, every `--context ./ProjectTalamh` call uses the cached index for faster, smarter file selection.

---

## Part 3: Custom Config Bank — The Game's Intelligence Layer

The default 15 agent configs (backend-api, frontend-component, etc.) are for web dev. For a Unity game, we create **game-specific configs** that the scoring system will pick over the defaults.

### 3.1 Agent Config: Combat Specialist

```json
// config-bank/agents/unity-combat.json
{
  "id": "unity-combat",
  "name": "Unity Combat Systems Engineer",
  "version": "1.0.0",
  "domains": ["combat", "physics", "animation", "input"],
  "stack": ["unity", "csharp", "rigidbody2d", "animator", "inputsystem"],
  "taskTypes": ["implementation", "refactor", "debug"],
  "systemPromptTemplate": "You are an expert Unity C# developer specializing in 2D action combat systems for beat 'em up games.\n\nYou build: combo state machines, hitbox systems, deflect/parry timing windows, juggle physics, wall bounce mechanics, and fighting-game-precision input buffering.\n\nRules:\n- All public APIs must match the interface contracts in Shared/Interfaces/\n- Use ScriptableObjects for attack data — never hardcode timing values\n- Use Animation Events for hitbox activation, not Update() checks\n- Input buffering: ~6 frames (0.1s at 60fps)\n- Use Rigidbody2D for knockback/launch, not transform.position\n- All timing values (deflect windows, i-frames) configurable via Inspector\n- No singletons — use [SerializeField] dependency injection\n- Comment WHY, not WHAT\n\nTask: {{task}}\n\nConstraints: {{constraints}}\n\nContext: {{context}}",
  "contextStrategy": "combat",
  "outputFormat": "C# source files (.cs) for Unity. MonoBehaviours for components, plain C# classes for logic, ScriptableObjects for data.",
  "qualityHistory": []
}
```

### 3.2 Agent Config: Roguelite Systems

```json
// config-bank/agents/unity-roguelite.json
{
  "id": "unity-roguelite",
  "name": "Roguelite Systems Architect",
  "version": "1.0.0",
  "domains": ["roguelite", "progression", "data", "ui"],
  "stack": ["unity", "csharp", "scriptableobject", "json", "eventsystem"],
  "taskTypes": ["implementation", "refactor", "debug"],
  "systemPromptTemplate": "You are an expert Unity C# developer specializing in roguelite progression systems (Hades/Dead Cells style).\n\nYou build: buff stacking systems, elemental ritual families, meta-progression trees, build-crafting synergy calculators, save/load persistence, and reward selection UIs.\n\nRules:\n- Subscribe to ICombatEvents — never reference combat MonoBehaviours directly\n- All ritual/trinket/inspiration data as ScriptableObjects\n- Ritual stacking is multiplicative (not additive) for Ritual Power\n- Twin Rituals require both elemental families present — enforce in code\n- Save/Load via JSON to Application.persistentDataPath\n- All currency modifications through CurrencyManager with events\n- No singletons — use SO-based event channels\n\nTask: {{task}}\n\nConstraints: {{constraints}}\n\nContext: {{context}}",
  "contextStrategy": "roguelite",
  "outputFormat": "C# source files (.cs) for Unity. ScriptableObject definitions, event channels, plain C# calculators for testability.",
  "qualityHistory": []
}
```

### 3.3 Agent Config: World Builder

```json
// config-bank/agents/unity-world.json
{
  "id": "unity-world",
  "name": "World & Content Builder",
  "version": "1.0.0",
  "domains": ["ai", "world", "enemies", "ui", "camera"],
  "stack": ["unity", "csharp", "animator", "statemachine", "canvas", "cinemachine"],
  "taskTypes": ["implementation", "refactor", "debug"],
  "systemPromptTemplate": "You are an expert Unity C# developer specializing in 2D game world systems.\n\nYou build: enemy AI (state machines/behavior trees), wave management, boss phase systems, branching path navigation, camera controllers, and HUD/UI.\n\nRules:\n- All enemies implement IDamageable and IAttacker interfaces from Shared/Interfaces/\n- Enemy attacks define TelegraphType (Normal vs Unstoppable) in AttackData\n- Boss AI uses explicit Phase state machine with transition conditions\n- Wave manager uses 'level bounds' (camera stops until wave cleared)\n- Branching path data as ScriptableObjects\n- Fire IRunProgressionEvents on area/boss/island completion\n- Health bars: world-space canvas. HUD: screen-space overlay.\n\nTask: {{task}}\n\nConstraints: {{constraints}}\n\nContext: {{context}}",
  "contextStrategy": "world",
  "outputFormat": "C# source files (.cs) for Unity. State machine implementations, ScriptableObject configs for enemy data, prefab setup instructions.",
  "qualityHistory": []
}
```

### 3.4 Context Strategies — File Prioritization per Domain

```json
// config-bank/strategies/combat.json
{
  "id": "combat",
  "name": "Combat Strategy",
  "domains": ["combat", "physics", "animation", "input"],
  "priorityPatterns": [
    "**/Combat/**",
    "**/Shared/Interfaces/**",
    "**/Shared/Data/**",
    "**/Shared/Enums/**",
    "**/ScriptableObjects/Attacks/**"
  ],
  "excludePatterns": [
    "**/Roguelite/**",
    "**/World/**",
    "**/UI/**",
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
// config-bank/strategies/roguelite.json
{
  "id": "roguelite",
  "name": "Roguelite Strategy",
  "domains": ["roguelite", "progression", "data"],
  "priorityPatterns": [
    "**/Roguelite/**",
    "**/Shared/Interfaces/**",
    "**/Shared/Data/**",
    "**/Shared/Enums/**",
    "**/ScriptableObjects/Rituals/**",
    "**/ScriptableObjects/Trinkets/**"
  ],
  "excludePatterns": [
    "**/Combat/**",
    "**/World/**",
    "**/*.unity",
    "**/*.meta",
    "**/Animations/**",
    "**/Prefabs/**"
  ],
  "defaultRelevance": 0.2
}
```

```json
// config-bank/strategies/world.json
{
  "id": "world",
  "name": "World Strategy",
  "domains": ["ai", "world", "enemies", "camera", "ui"],
  "priorityPatterns": [
    "**/World/**",
    "**/Shared/Interfaces/**",
    "**/Shared/Data/**",
    "**/Shared/Enums/**",
    "**/ScriptableObjects/Enemies/**",
    "**/ScriptableObjects/Islands/**"
  ],
  "excludePatterns": [
    "**/Combat/**",
    "**/Roguelite/**",
    "**/*.unity",
    "**/*.meta",
    "**/Animations/**"
  ],
  "defaultRelevance": 0.2
}
```

### 3.5 How Config Scoring Works

When Dev 1 runs:
```bash
agent-pilot "Implement the Deflect system with i-frame timing and parry window" --context ./ProjectTalamh
```

The analyzer detects `domains: [combat, input]`, `stack: [unity, csharp]`, `taskType: implementation`.

Config scoring (weighted: domain 0.4, stack 0.35, type 0.25):

| Config | Domain (0.4) | Stack (0.35) | Type (0.25) | Total |
|--------|-------------|-------------|-------------|-------|
| **unity-combat** | **0.40** | **0.35** | **0.25** | **1.00** ← Selected |
| unity-world | 0.10 | 0.25 | 0.25 | 0.60 |
| unity-roguelite | 0.00 | 0.25 | 0.25 | 0.50 |
| backend-api | 0.00 | 0.00 | 0.25 | 0.25 |

The `combat` context strategy then kicks in:
- `Assets/Scripts/Combat/**` → FULL (complete content)
- `Assets/Scripts/Shared/Interfaces/**` → FULL
- `Assets/Scripts/Roguelite/**` → SKIP (excluded)
- `Assets/Scripts/World/**` → SKIP (excluded)

**The agent never sees roguelite or world code.** That's the point.

---

## Part 4: The Planner — Decomposing the Game into Tasks

### 4.1 Project-Level Planning

Use `/plan-project` (or the programmatic API) to break the entire game into phased tasks:

```bash
agent-pilot plan-project "Build a 2D side-scrolling beat em up roguelite in Unity. \
  Core pillars: combat depth through defensive mechanics (deflect, clash, punish), \
  roguelite build-crafting with elemental ritual families, \
  replayable branching runs with hand-crafted maps. \
  4 playable characters, 8 ritual families, 4 islands, boss fights, \
  local + online 2-player co-op."
```

Agent Pilot's **planner module** does this without LLM calls (deterministic decomposition):

1. **`decomposeTasks`** → Keyword analysis + domain templates → 35-50 tasks
2. **`buildDependencyGraph`** → DAG construction with cycle detection
3. **`getParallelGroups`** → Tasks that can run simultaneously (one per dev)
4. **`clusterTasks`** (context affinity) → Groups tasks by shared domains (0.4), files (0.35), stack (0.25)
5. **`budgetPhases`** → Token allocation per phase with handoff overhead (~1K per phase transition)

### 4.2 Expected Phase Output

```
Phase 1: Foundation (3 parallel groups)
  ├── Group A (Dev 1): COMBAT-001 CharacterController2D, COMBAT-002 InputBuffer
  ├── Group B (Dev 2): RITUAL-001 RitualData SO, RITUAL-002 Enums/SharedTypes
  └── Group C (Dev 3): WORLD-001 WaveManager, WORLD-002 CameraController

Phase 2: Core Mechanics
  ├── Group A: COMBAT-003 ComboStateMachine, COMBAT-004 HitboxManager
  ├── Group B: RITUAL-003 RitualSystem, RITUAL-004 StackCalculator
  └── Group C: WORLD-003 EnemyBase(IDamageable), WORLD-004 BasicEnemyAI

Phase 3: Defensive Systems + Build Crafting
  ├── Group A: COMBAT-005 Deflect/Clash, COMBAT-006 PressureSystem
  ├── Group B: RITUAL-005 TrinketSystem, RITUAL-006 RewardSelectionUI
  └── Group C: WORLD-005 BossAI Framework, WORLD-006 BranchingPaths

Phase 4: Integration + Content
  ├── Group A: COMBAT-007 WallBounce, COMBAT-008 AirJuggles
  ├── Group B: RITUAL-007 MetaProgression, RITUAL-008 SaveLoad
  └── Group C: WORLD-007 QuestSystem, WORLD-008 HUD

Phase 5: Polish + Co-op
  ├── All: Integration testing, co-op, VFX, balancing
```

### 4.3 Context Affinity Clustering

The planner's `clusterTasks` uses weighted affinity scoring:

```
Affinity(task_A, task_B) = 
    (shared_domains × 0.40) + (shared_files × 0.35) + (shared_stack × 0.25)
```

This naturally clusters combat tasks together, roguelite tasks together, etc. — matching our 3-dev split. Tasks that span domains (like "connect ritual buffs to combat damage") get flagged as **cross-domain integration tasks** that need context from both strategies.

---

## Part 5: Execution — Daily Developer Workflow

### 5.1 Dev 1 (Combat) — Typical Session

```bash
# Morning: execute today's task
agent-pilot "Implement the Deflect system. When player dashes with correct \
  timing against an incoming non-Unstoppable attack, open a Punish window. \
  Deflect window: 0-150ms after dash start (configurable). \
  Return DamageResponse.Deflected which the PressureSystem reads as punish damage." \
  --context ./ProjectTalamh \
  --budget 10000 \
  --verbose

# Agent Pilot:
# [1/8] ANALYZE    domains=[combat,input] stack=[unity,csharp] type=implementation complexity=medium
# [2/8] BUILD      config=unity-combat (score: 1.00) strategy=combat
# [3/8] CONTEXT    12 files scored → 4 FULL, 2 SUMMARY, 6 SKIP → 4,800 tokens
# [4/8] EXECUTE    running in clean context window...
# [5/8] CAPTURE    2 files produced (DefenseSystem.cs, DefenseSystem.test.cs)
# [6/8] TEST       5/5 checks passed
# [7/8] DOCUMENT   task-spec.md, agent-config.md, summary.md
# [8/8] REPORT     Quality: 88/100 | Tokens: 6,200/10,000 | Time: 12.3s

# Review the output
cat output/task-*/files/DefenseSystem.cs

# If it looks good, copy to Unity project
cp output/task-*/files/*.cs Assets/Scripts/Combat/

# If it needs tweaks, iterate:
agent-pilot "Fix DefenseSystem.cs: the deflect window should use Time.time \
  not Time.deltaTime for comparison. Also add a [Header('Deflect')] attribute \
  group in Inspector." \
  --context ./ProjectTalamh --budget 5000
```

### 5.2 Dev 2 (Roguelite) — Typical Session

```bash
# Execute ritual stacking task
agent-pilot "Implement RitualStackCalculator. Rules: \
  - Rituals of same mechanic stack when acquired multiple times \
  - Level 1 = base, Level 2 = 1.5x base, Level 3 = 2.0x base \
  - Multiple different rituals buffing same mechanic: 2 rituals = 1.5x, 3 = 2.0x \
  - Ritual Power (from Trinkets) is MULTIPLICATIVE with itself: \
    two +10% trinkets = 1.1 * 1.1 = 1.21x, NOT 1.2x \
  - Ritual Power multiplies FINAL value after level + stacking \
  - Make it a plain C# class, not MonoBehaviour, for unit testing" \
  --context ./ProjectTalamh --budget 8000
```

Notice: the task prompt includes **concrete numeric examples** (`1.1 * 1.1 = 1.21x, NOT 1.2x`). This is critical — Agent Pilot will pass this to the agent in its clean context window, preventing the common "multiplicative vs additive" misinterpretation.

### 5.3 Dev 3 (World) — Typical Session

```bash
# Execute boss AI framework
agent-pilot "Implement a BossAI base class with: \
  - Phase system (enum BossPhase, configurable transition thresholds by HP%) \
  - Each phase has its own attack pattern list (ScriptableObject) \
  - Punish windows after big attacks (duration configurable per attack) \
  - Unstoppable attacks marked with TelegraphType.Unstoppable in AttackData \
  - Stun state integration: when PressureSystem triggers stun, boss enters StunnedPhase \
  - Camera zoom-in on stun (fire event, don't reference camera directly) \
  - Must implement IDamageable and IAttacker from Shared/Interfaces" \
  --context ./ProjectTalamh --budget 12000
```

### 5.4 Progressive Compression in Action

When Dev 3's task runs, the `world` context strategy scores project files:

```
File Scoring for "Boss AI Framework" task:
──────────────────────────────────────────
Assets/Scripts/World/EnemyBase.cs          → relevance: 0.95 → FULL (2,100 tokens)
Assets/Scripts/Shared/Interfaces/IDamageable.cs → relevance: 0.95 → FULL (400 tokens)
Assets/Scripts/Shared/Interfaces/IAttacker.cs   → relevance: 0.95 → FULL (350 tokens)
Assets/Scripts/Shared/Data/AttackData.cs   → relevance: 0.90 → FULL (300 tokens)
Assets/Scripts/World/BasicEnemyAI.cs       → relevance: 0.80 → FULL (1,500 tokens)
Assets/Scripts/Shared/Enums/TelegraphType.cs → relevance: 0.85 → FULL (50 tokens)
Assets/Scripts/World/WaveManager.cs        → relevance: 0.40 → SUMMARY (200 tokens)
Assets/Scripts/Combat/PressureSystem.cs    → relevance: 0.35 → REFERENCE (20 tokens)
Assets/Scripts/Combat/ComboSystem.cs       → relevance: 0.10 → SKIP
Assets/Scripts/Roguelite/RitualSystem.cs   → relevance: 0.05 → SKIP
                                            ───────────────────
                              Total context: 4,920 tokens ✓ within 7K budget
```

**Notice:** `PressureSystem.cs` gets REFERENCE (just the path) because the boss needs to know it exists and fires stun events, but doesn't need the implementation. `ComboSystem.cs` is completely excluded — irrelevant to boss AI.

---

## Part 6: Phase Execution with the Orchestrator

### 6.1 Running a Full Phase

Instead of task-by-task, use `/execute-phase` to batch all tasks in a phase:

```bash
agent-pilot execute-phase 2
```

The **orchestrator module** handles:
- **`PhaseRunner`** → Dependency-aware batching, runs independent tasks in parallel
- **`ArtifactRegistry`** → Stores outputs with tags, relevance scores, persists to disk
- **`HandoffManager`** → Generates ~1K token summaries for the next phase

### 6.2 Quality Gates

Each phase has automatic quality gates:
- **Minimum 60%** of tasks must succeed
- **Minimum 50** average quality score

```
Phase 2: Core Mechanics
────────────────────────────────────────
Tasks: 6 passed, 0 failed, 6 total
Quality gate: PASSED — 100% pass rate, 81 avg quality
Time: 67.4s

  [OK] COMBAT-003 (ComboStateMachine):     quality=85
  [OK] COMBAT-004 (HitboxManager):         quality=78
  [OK] RITUAL-003 (RitualSystem):           quality=82
  [OK] RITUAL-004 (StackCalculator):        quality=88
  [OK] WORLD-003 (EnemyBase):              quality=75
  [OK] WORLD-004 (BasicEnemyAI):           quality=79
```

If a phase **fails** the quality gate, you know before moving on. Fix the failing tasks, re-run, proceed.

### 6.3 Handoff Between Phases

The `HandoffManager` generates a ~1K token summary of what Phase 2 produced, which gets injected as context for Phase 3 agents. This is how later tasks know about earlier decisions without getting the full implementation code:

```
Phase 2 Summary (for Phase 3 context):
- ComboSystem: State machine with ComboNode tree, supports Strike/Skill/Arcana branches,
  dash-cancel on hit, input buffer of 0.1s. Key class: ComboSystem.cs
- HitboxManager: Animation Event driven, activates/deactivates colliders per attack.
  Key class: HitboxManager.cs
- RitualSystem: Subscribes to ICombatEvents, manages active rituals per run,
  fires ritual effects via RitualTrigger→RitualEffect pipeline. Key class: RitualSystem.cs
- RitualStackCalculator: Plain C# class, multiplicative stacking.
  Key class: RitualStackCalculator.cs
- EnemyBase: Implements IDamageable + IAttacker, handles HP, pressure, knockback.
  Key class: EnemyBase.cs
- BasicEnemyAI: Idle→Patrol→Chase→Attack→HitReact→Death state machine.
  Key class: BasicEnemyAI.cs
```

Phase 3 agents get this summary (~800 tokens) instead of 6 full source files (~12K tokens). **Context compression across time, not just across files.**

---

## Part 7: The Flywheel — Continuous Improvement

### 7.1 Capture Learnings After Each Sprint

```bash
agent-pilot capture-learnings
```

This updates the `qualityHistory` array on each config. Over time:

```json
// After 20 combat tasks:
{
  "id": "unity-combat",
  "qualityHistory": [88, 92, 75, 85, 90, 82, 95, 78, 88, 91, ...]
  // Average: 86.4 — this config is performing well for combat tasks
}
```

Configs that consistently underperform for certain task types can be tuned: adjust the system prompt, refine the context strategy patterns, or create a more specialized config.

### 7.2 Adding New Configs as the Project Evolves

As you discover recurring task patterns, create specialized configs:

```json
// config-bank/agents/unity-combat-feel.json
// For "it works but doesn't feel right" tuning tasks
{
  "id": "unity-combat-feel",
  "name": "Combat Feel Tuner",
  "domains": ["combat", "animation", "physics"],
  "stack": ["unity", "csharp", "dotween"],
  "taskTypes": ["refactor"],
  "systemPromptTemplate": "You are a game feel specialist. You tune timing values, \
    easing curves, screen shake, hitstop, and juice to make combat FEEL impactful. \
    You think in frames (60fps) not seconds. You know that 2-3 frames of hitstop \
    on heavy hits makes them feel powerful. You add DOTween screen shake on impacts.\n\n...",
  "contextStrategy": "combat",
  "outputFormat": "Modified C# files with adjusted timing values, added juice effects.",
  "qualityHistory": []
}
```

**Auto-discovery:** Just drop the JSON in `config-bank/agents/` — the config bank auto-discovers it. No registration needed.

---

## Part 8: Full Orchestration — /build-app

For a new sprint or major feature, use the full orchestration command:

```bash
agent-pilot build-app "Project Talamh Sprint 3" unity-game \
  "Implement defensive combat systems (Deflect, Clash, Punish, Pressure/Stun), \
   the Ritual trigger→effect pipeline with Fire and Lightning families, \
   and Boss AI with phases and punish windows"
```

This chains: **plan → configure → execute all phases → capture learnings**

The orchestrator runs tasks respecting the dependency graph, manages artifacts, enforces quality gates, and produces a full audit trail.

---

## Part 9: Cross-Domain Integration Tasks

Some tasks span domains. These are the trickiest because they need context from multiple strategies.

### 9.1 Example: "Connect Ritual Buffs to Combat Damage"

This needs both combat code (damage calculation) AND roguelite code (buff provider).

**Solution:** Create a temporary integration config:

```json
// config-bank/agents/unity-integration.json
{
  "id": "unity-integration",
  "name": "Cross-Domain Integrator",
  "domains": ["combat", "roguelite", "world"],
  "stack": ["unity", "csharp"],
  "taskTypes": ["implementation", "refactor"],
  "systemPromptTemplate": "You integrate systems across domain boundaries in a Unity game project. \
    You work ONLY through the interfaces defined in Shared/Interfaces/. \
    You never add direct references between Combat/, Roguelite/, and World/ folders. \
    Your job is to wire events, inject dependencies, and ensure the contracts work at runtime.\n\n...",
  "contextStrategy": "fullstack",
  "outputFormat": "Integration wiring code, dependency injection setup, integration tests.",
  "qualityHistory": []
}
```

Use the `fullstack` strategy (balanced across all file types) but keep the budget tight so compression keeps it focused.

### 9.2 Integration Sync Points

All 3 devs coordinate at these moments:

| Event | What Happens | Agent Pilot Command |
|-------|-------------|-------------------|
| Interface change | All 3 review. Update Shared/Interfaces/ | Manual review + re-run `scan-repo` |
| Phase complete | Merge branches, verify together | `execute-phase N` + quality gate check |
| Integration build | Test all systems working together | `agent-pilot "Integration test: ..." --context .` with `unity-integration` config |
| Sprint review | Demo, log lessons, update configs | `capture-learnings` |

---

## Part 10: Programmatic API for Custom Workflows

For more control, use Agent Pilot as a library:

```typescript
import { 
  executePipeline, 
  analyzeTask, 
  buildAgent, 
  decomposeTasks, 
  buildDependencyGraph,
  getParallelGroups,
  budgetPhases 
} from 'agent-pilot';

// Analyze a task without executing (for planning)
const analysis = await analyzeTask(
  "Implement wall bounce physics for combat juggles",
  "./ProjectTalamh"
);
console.log(analysis.domains);    // ["combat", "physics"]
console.log(analysis.complexity); // "medium"

// Build the agent to inspect config selection (without executing)
const agent = buildAgent(analysis, projectFiles, 8000);
console.log(agent.configSources); // ["unity-combat"]
console.log(agent.contextFiles);  // Only combat + shared interface files

// Execute the full pipeline
const metrics = await executePipeline(
  "Implement wall bounce physics for combat juggles", 
  {
    contextPath: './ProjectTalamh',
    budget: 10000,
    verbose: true
  }
);
console.log(metrics.qualityScore);  // 85
console.log(metrics.filesCreated);  // ["WallBounce.cs"]
```

### Custom Batch Runner (Run All Dev 1 Tasks)

```typescript
import { executePipeline } from 'agent-pilot';

const dev1Tasks = [
  "Implement CharacterController2D with 8-dir movement, jump, dash, gravity",
  "Implement InputBufferSystem with 6-frame buffer window",
  "Implement basic Strike combo chain: 3 hits + finisher, animation-driven",
  "Create AttackData ScriptableObject: damage, knockback, launchForce, animClip, timing",
];

for (const task of dev1Tasks) {
  const metrics = await executePipeline(task, {
    contextPath: './ProjectTalamh',
    budget: 10000,
    verbose: true,
  });
  
  console.log(`${task.slice(0, 50)}... → Quality: ${metrics.qualityScore}`);
  
  if (metrics.qualityScore < 70) {
    console.warn(`⚠️ Low quality — review before integrating`);
  }
}
```

---

## Part 11: Quick Reference

```
╔══════════════════════════════════════════════════════════════════╗
║              AGENT PILOT × PROJECT TALAMH                       ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  DAILY COMMANDS                                                 ║
║  ─────────────                                                  ║
║  agent-pilot "<task>" --context . --budget 10000                ║
║  agent-pilot execute-phase N                                    ║
║  agent-pilot scan-repo ./ProjectTalamh                          ║
║  agent-pilot capture-learnings                                  ║
║                                                                  ║
║  CUSTOM CONFIGS (in config-bank/agents/)                        ║
║  ──────────────                                                 ║
║  unity-combat.json      → Dev 1 tasks (combo, hitbox, defense)  ║
║  unity-roguelite.json   → Dev 2 tasks (rituals, meta, save)     ║
║  unity-world.json       → Dev 3 tasks (enemies, bosses, paths)  ║
║  unity-integration.json → Cross-domain wiring tasks              ║
║  unity-combat-feel.json → Game feel tuning tasks                 ║
║                                                                  ║
║  CONTEXT STRATEGIES (in config-bank/strategies/)                 ║
║  ─────────────────                                              ║
║  combat.json    → Prioritizes Combat/ + Shared/Interfaces/      ║
║  roguelite.json → Prioritizes Roguelite/ + Shared/Interfaces/   ║
║  world.json     → Prioritizes World/ + Shared/Interfaces/        ║
║                                                                  ║
║  TOKEN BUDGET                                                    ║
║  ────────────                                                   ║
║  Default: 10K per task (7K agent + 2K test + 1K docs)           ║
║  Hard ceiling: 15K per agent (NEVER exceeded)                   ║
║  Compression: FULL → SUMMARY → REFERENCE → SKIP                ║
║                                                                  ║
║  GOLDEN RULE                                                     ║
║  An agent that knows less, produces better output.               ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```
