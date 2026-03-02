# Agent Pilot Operational Guide
## Building "Project Talamh" — An Absolum-Style Beat 'Em Up Roguelite in Unity

### For a 3-Developer Team Using the Agent Pilot Context Orchestration Method

---

## Part 1: Agent Pilot Philosophy Recap

Agent Pilot is a **context orchestration system** that breaks complex tasks into focused, context-minimal operations executed by purpose-built agents. The core principles:

1. **Context is the bottleneck.** LLMs perform better with focused, relevant context than with everything dumped in. Each agent gets ONLY what it needs.
2. **The 8-step pipeline** takes a complex task → analyzes it → builds purpose-specific agents → executes them → validates output.
3. **Agents are disposable and purpose-built.** You don't reuse a generic "coding agent" — you spin up an agent with a precise system prompt, precise context docs, and a precise task scope.
4. **Humans are Product Owners.** Devs define WHAT, agents execute HOW, devs validate output.

### The 8-Step Pipeline Applied to Game Dev

```
1. TASK INTAKE        → "Implement the Deflect system for player combat"
2. TASK ANALYSIS      → Decompose into sub-tasks, identify dependencies
3. CONTEXT ASSEMBLY   → Pull ONLY relevant docs: interface contracts, combat design spec, existing CharacterController
4. AGENT DESIGN       → Build a system prompt for a "Unity Combat Systems Engineer" agent
5. AGENT EXECUTION    → Agent writes code with minimal, focused context
6. OUTPUT VALIDATION  → Does it compile? Does it follow the interface contract? Does it feel right in-game?
7. INTEGRATION        → Merge into the shared codebase
8. FEEDBACK LOOP      → Update project docs, note what worked, refine for next task
```

---

## Part 2: Project Setup — Before Anyone Writes Code

### 2.1 The Shared Knowledge Base

Before any agent can be effective, the team needs a **Project Knowledge Base (PKB)** — a set of markdown docs that agents will pull context from. This lives in the repo root:

```
ProjectTalamh/
├── .agentpilot/                     ← Agent Pilot configuration
│   ├── project-brief.md             ← High-level game vision (ALL agents read this)
│   ├── architecture.md              ← System architecture, folder structure
│   ├── interface-contracts.md       ← The shared API between all 3 dev domains
│   ├── design-specs/
│   │   ├── combat-design.md         ← Absolum reverse-engineering: combat mechanics
│   │   ├── roguelite-design.md      ← Rituals, meta-progression, build-crafting
│   │   ├── world-design.md          ← Islands, branching paths, enemy types, bosses
│   │   └── art-pipeline.md          ← Animation state requirements, VFX specs
│   ├── agent-templates/             ← Reusable agent system prompts
│   │   ├── combat-engineer.md
│   │   ├── roguelite-architect.md
│   │   ├── world-builder.md
│   │   ├── unity-reviewer.md
│   │   └── test-writer.md
│   └── task-log.md                  ← Running log of completed agent tasks + outcomes
├── Assets/                          ← Unity project
│   └── ...
└── Docs/
    └── absolum-reverse-engineering.md  ← The research doc from our earlier session
```

### 2.2 Project Brief (project-brief.md)

This is the ONE document every agent gets in its context, regardless of its role. Keep it tight — under 500 words:

```markdown
# Project Talamh — Project Brief

## What We're Building
A 2D side-scrolling beat 'em up with roguelite progression, inspired by Absolum.
Hand-drawn-style characters and environments. Dark fantasy setting.

## Core Pillars
1. **Combat depth through defensive mechanics** — Deflect, Clash, Punish
   are the skill expression, not just combo chains
2. **Roguelite build-crafting** — Elemental ritual families that stack and synergize
3. **Replayable branching runs** — Fixed hand-crafted maps with path choices

## Tech Stack
- Unity 2022 LTS (2D URP)
- C# with ScriptableObject-driven data architecture
- Input System package (new)
- DOTween for juice
- Fishnet or Mirror for co-op (TBD)

## Team
- Dev 1 (Combat): Owns combat sandbox, character controllers, hitbox systems
- Dev 2 (Roguelite): Owns rituals, trinkets, meta-progression, hub
- Dev 3 (World): Owns wave management, enemy AI, bosses, UI, branching paths

## Non-Negotiable Conventions
- All cross-domain communication through interfaces defined in interface-contracts.md
- ScriptableObjects for ALL data (attacks, rituals, enemies, trinkets)
- No MonoBehaviour singletons — use dependency injection or SO-based event channels
- Animator Override Controllers for character variants
- Animation Events for hitbox timing, VFX, SFX
```

### 2.3 Interface Contracts (interface-contracts.md)

This is the **most critical document in the project.** It's what allows 3 devs (and their agents) to work in parallel without stepping on each other. All 3 devs co-author this BEFORE writing gameplay code:

```markdown
# Interface Contracts — Project Talamh

All cross-domain communication MUST go through these interfaces.
Do NOT reference concrete implementations across domain boundaries.

## Combat → Roguelite (Dev 1 fires, Dev 2 subscribes)

```csharp
public interface ICombatEvents
{
    event System.Action<StrikeEventData> OnStrike;
    event System.Action<DashEventData> OnDash;
    event System.Action<DeflectEventData> OnDeflect;
    event System.Action<ClashEventData> OnClash;
    event System.Action<PunishEventData> OnPunish;
    event System.Action<KillEventData> OnKill;
    event System.Action<FinisherEventData> OnFinisher;
    event System.Action<ArcanaEventData> OnArcana;
    event System.Action<JumpEventData> OnJump;
    event System.Action<DodgeEventData> OnDodge;
}
```

## Roguelite → Combat (Dev 2 provides, Dev 1 queries)

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

## Combat → World (Dev 1 defines, Dev 3 implements on enemies)

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

public interface IAttacker
{
    AttackData CurrentAttack { get; }
    bool IsCurrentAttackUnstoppable { get; }
    TelegraphType CurrentTelegraphType { get; }
    float PunishWindowDuration { get; }
    bool IsInPunishableState { get; }
}
```

## World → Roguelite (Dev 3 fires, Dev 2 subscribes)

```csharp
public interface IRunProgressionEvents
{
    event System.Action<AreaClearedData> OnAreaCleared;
    event System.Action<BossDefeatedData> OnBossDefeated;
    event System.Action<IslandCompletedData> OnIslandCompleted;
    event System.Action<ShopEnteredData> OnShopEntered;
    event System.Action OnRunStarted;
    event System.Action<RunEndData> OnRunEnded;
}
```

## Shared Data Structures

```csharp
[CreateAssetMenu] public class AttackData : ScriptableObject { ... }
public struct DamagePacket { DamageType type; float amount; bool isPunish; ... }
public enum DamageType { Physical, Fire, Lightning, Water, Thorn, Gale, Time, Cosmic }
public enum RitualTrigger { OnStrike, OnDash, OnDeflect, OnClash, OnFinisher, OnKill, OnArcana, OnJump, OnDodge, OnTakeDamage }
public enum TelegraphType { Normal, Unstoppable }
```
```

---

## Part 3: Agent Templates — The Toolbox

Each dev creates and refines agent templates that they reuse across tasks. These are system prompts stored in `.agentpilot/agent-templates/`.

### 3.1 Combat Engineer Agent (Dev 1)

```markdown
# Agent: Unity Combat Systems Engineer

## Role
You are an expert Unity C# developer specializing in 2D action combat systems.
You build beat 'em up mechanics: combo state machines, hitbox systems,
deflect/parry timing, juggle physics, and fighting-game-precision input handling.

## Context You Will Receive
- The interface contracts (always)
- The specific task description
- Any relevant existing code files
- The combat design spec (when needed)

## Rules
- All public combat APIs must match the interface contracts exactly
- Use ScriptableObjects for attack data — never hardcode values
- Use Animation Events for hitbox activation frames
- Input buffering: store inputs for ~6 frames (0.1s at 60fps)
- Physics: use Rigidbody2D for knockback/launch, not transform manipulation
- All timing values (deflect window, i-frames, etc.) must be configurable in Inspector
- Comment WHY, not WHAT
- No singletons — accept dependencies via [SerializeField] or constructor injection

## Output Format
- Complete .cs files ready to drop into the project
- Brief explanation of design decisions
- Any questions about interfaces or edge cases
```

### 3.2 Roguelite Systems Architect Agent (Dev 2)

```markdown
# Agent: Roguelite Systems Architect

## Role
You are an expert Unity C# developer specializing in roguelite progression systems.
You build Hades/Dead Cells-style buff stacking, build-crafting, and meta-progression.
You think in ScriptableObject architectures and event-driven designs.

## Context You Will Receive
- The interface contracts (always)
- The specific task description
- The roguelite design spec (ritual families, synergies, meta-progression)
- Any relevant existing code files

## Rules
- Subscribe to ICombatEvents — never directly reference combat MonoBehaviours
- All ritual/trinket/inspiration data as ScriptableObjects
- Ritual stacking math must be multiplicative (not additive) for Ritual Power
- Twin Rituals require presence of both elemental families — enforce this in code
- Save/Load via JSON serialization to Application.persistentDataPath
- Reward selection UI must support 2 or 3 options (configurable via Soul Tree upgrade)
- All currency modifications go through a central CurrencyManager with events
- No direct scene references — use SO-based event channels for run progression signals

## Output Format
- Complete .cs files + ScriptableObject definitions
- Data flow diagrams when building new subsystems
- Brief explanation of stacking/synergy math
```

### 3.3 World Builder Agent (Dev 3)

```markdown
# Agent: World & Content Builder

## Role
You are an expert Unity C# developer specializing in 2D game world systems.
You build enemy AI (behavior trees), wave management, boss phase systems,
branching level navigation, and HUD/UI using Unity's UI Toolkit or UGUI.

## Context You Will Receive
- The interface contracts (always)
- The specific task description
- The world design spec (islands, enemies, bosses, paths)
- Any relevant existing code files

## Rules
- All enemies implement IDamageable and IAttacker interfaces
- Enemy attacks define TelegraphType (Normal vs Unstoppable) in their AttackData
- Boss AI uses a state machine with explicit Phase transitions
- Wave manager uses "level bounds" (camera stops until wave cleared)
- Branching path data as ScriptableObjects (PathNodeData)
- Camera follows player with configurable leading, zoom-on-stun, and co-op framing
- UI: health bars use world-space canvas, HUD uses screen-space overlay
- Fire IRunProgressionEvents when areas/bosses/islands are completed

## Output Format
- Complete .cs files
- Enemy behavior descriptions (state diagrams for bosses)
- Prefab setup instructions when relevant
```

### 3.4 Shared Utility Agents

```markdown
# Agent: Unity Code Reviewer

## Role
You review Unity C# code for correctness, performance, and adherence to
Project Talamh's conventions.

## You Check For
- Interface contract compliance
- No cross-domain concrete references
- ScriptableObject usage for data
- No singletons
- Proper use of Animation Events (not Update-based hitbox checks)
- Memory: no per-frame allocations, object pooling for projectiles/effects
- Physics: FixedUpdate for Rigidbody, Update for input
- Null safety on serialized fields
```

```markdown
# Agent: Unit Test Writer

## Role
You write NUnit tests for Unity C# code. You focus on testing logic
in isolation by mocking interfaces.

## Rules
- Use NUnit + NSubstitute for mocking
- Test combat math (damage * buff multiplier * punish multiplier)
- Test ritual stacking math (multiplicative scaling)
- Test pressure meter thresholds
- Test combo state machine transitions
- Never test MonoBehaviour lifecycle — test extracted logic classes
```

---

## Part 4: Daily Workflow — How a Dev Uses Agent Pilot

### 4.1 The Task Lifecycle

Here's how Dev 2 (Roguelite) would tackle "Implement the Ritual Stacking System":

#### Step 1: TASK INTAKE
Dev 2 creates a task in the task log:
```markdown
## Task: RITUAL-004 — Ritual Stacking System
**Owner:** Dev 2
**Status:** In Progress
**Depends on:** RITUAL-001 (RitualData SO), RITUAL-002 (RitualFamily enum)
**Description:** When a player has multiple rituals that enhance the same
mechanic (e.g., two rituals that both invoke tidal waves), their power
should stack. Level 2 = 50% stronger, Level 3 = 100% stronger.
Ritual Power (from trinkets) is multiplicative on top.
```

#### Step 2: TASK ANALYSIS — Break It Down
Before touching an agent, Dev 2 decomposes into atomic sub-tasks:

```
RITUAL-004a: RitualInstance class (tracks level, stacking group)
RITUAL-004b: Stacking calculator (group same-mechanic rituals, compute combined power)
RITUAL-004c: Ritual Power modifier integration (multiplicative from Trinkets)
RITUAL-004d: Unit tests for stacking math
RITUAL-004e: Integration with RitualSystem.OnCombatEvent pipeline
```

#### Step 3: CONTEXT ASSEMBLY — Minimal & Focused
For sub-task RITUAL-004b, the agent needs ONLY:
- `interface-contracts.md` (just the IBuffProvider section)
- `RitualData.cs` (the SO definition from RITUAL-001)
- `RitualFamily.cs` (the enum)
- The stacking rules from `roguelite-design.md` (just the relevant paragraph)

**NOT included:** Combat system code, enemy AI, wave manager, UI code, boss patterns — none of that is relevant. This is the context-minimal principle in action.

#### Step 4: AGENT DESIGN — Spin Up the Right Agent
Dev 2 uses the `roguelite-architect` template but adds a task-specific preamble:

```markdown
# Task-Specific Context

You are implementing the Ritual Stacking Calculator.

## Stacking Rules (from Absolum)
- Rituals of the same mechanic stack when acquired multiple times
- Level 1 = base power, Level 2 = 1.5x base, Level 3 = 2.0x base
- Having multiple DIFFERENT rituals that buff the same mechanic (e.g., two
  rituals that both invoke "Small Tidal Wave") increases damage: 
  2 rituals = 1.5x, 3 rituals = 2.0x
- Ritual Power (from Trinkets like Moxes) is MULTIPLICATIVE with itself:
  Two +10% Ritual Power trinkets = 1.1 * 1.1 = 1.21x, NOT 1.2x
- Ritual Power multiplies the FINAL value after level and stacking

## Existing Code You're Working With
[paste RitualData.cs]
[paste RitualFamily.cs]
[paste IBuffProvider interface section]

## Your Task
Implement a RitualStackCalculator class that:
1. Groups active rituals by shared mechanic
2. Computes effective power considering level + stacking + Ritual Power
3. Exposes a method: float GetEffectivePower(RitualMechanic mechanic)
4. Is a plain C# class (not MonoBehaviour) for testability
```

#### Step 5: AGENT EXECUTION
Run the agent. It produces `RitualStackCalculator.cs`.

#### Step 6: OUTPUT VALIDATION
Dev 2 checks:
- [ ] Does the math match the spec? (Level 3 = 2.0x, not 3.0x?)
- [ ] Is Ritual Power multiplicative with ITSELF? (1.1 * 1.1, not 1.1 + 0.1?)
- [ ] Is it a plain C# class? (Not accidentally a MonoBehaviour?)
- [ ] Does it depend only on RitualData and RitualFamily? (No combat imports?)

If there are issues → feed back to agent with corrections → iterate.

#### Step 7: INTEGRATION
- Drop into `Assets/Scripts/Roguelite/`
- Verify no compile errors
- Wire into RitualSystem.cs
- Push to branch `roguelite/ritual-stacking`

#### Step 8: FEEDBACK LOOP
Update task log:
```markdown
**Status:** Complete
**Agent used:** roguelite-architect + task-specific stacking context
**Iterations:** 2 (first pass had additive Ritual Power, corrected to multiplicative)
**Lesson:** Always include a concrete numeric example in the task spec to
prevent the agent from misinterpreting "multiplicative with itself"
```

### 4.2 Visualizing the Full Cycle

```
 Dev thinks about the task
         │
         ▼
 ┌─────────────────┐
 │  Decompose into  │
 │  atomic sub-tasks │
 └────────┬─────────┘
          │
          ▼
 ┌─────────────────┐
 │ Assemble MINIMAL │  ← Only the docs/code this specific sub-task needs
 │ context package   │
 └────────┬─────────┘
          │
          ▼
 ┌─────────────────┐
 │ Select/customize │  ← Pick from .agentpilot/agent-templates/
 │ agent template   │  ← Add task-specific preamble + context
 └────────┬─────────┘
          │
          ▼
 ┌─────────────────┐
 │ Execute agent    │  ← Claude Code, ChatGPT, Cursor, etc.
 │ (any LLM tool)  │
 └────────┬─────────┘
          │
          ▼
 ┌─────────────────┐
 │ Validate output  │  ← Compiles? Matches contract? Correct math?
 │                  │  ← If no → iterate with corrections
 └────────┬─────────┘
          │
          ▼
 ┌─────────────────┐
 │ Integrate +      │  ← Merge, update task log, note lessons
 │ Update docs     │
 └─────────────────┘
```

---

## Part 5: Parallel Execution — 3 Devs Working Simultaneously

### 5.1 The Golden Rule

> **Agents from different domains should NEVER need code from other domains in their context.**
> They only need the interface contracts.

This is why the interface contracts are written first. Dev 1's combat agent doesn't need to know how rituals stack. Dev 2's roguelite agent doesn't need to know how hitboxes work. Dev 3's enemy AI agent doesn't need to know how trinkets modify damage.

### 5.2 What Each Dev's Agent Context Looks Like

```
┌─────────────────────────────────────────────────────────────┐
│ Dev 1 (Combat) Agent Context                                │
│                                                             │
│ ✅ project-brief.md (always)                                │
│ ✅ interface-contracts.md (always)                           │
│ ✅ combat-design.md (when needed)                           │
│ ✅ Their own combat code files (relevant ones only)         │
│ ❌ Ritual system code                                       │
│ ❌ Enemy AI code                                            │
│ ❌ Wave manager code                                        │
│ ❌ UI code                                                  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Dev 2 (Roguelite) Agent Context                             │
│                                                             │
│ ✅ project-brief.md (always)                                │
│ ✅ interface-contracts.md (always)                           │
│ ✅ roguelite-design.md (when needed)                        │
│ ✅ Their own roguelite code files (relevant ones only)      │
│ ❌ Combat internals (combo state machine, hitbox logic)     │
│ ❌ Enemy AI code                                            │
│ ❌ Wave manager code                                        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Dev 3 (World) Agent Context                                 │
│                                                             │
│ ✅ project-brief.md (always)                                │
│ ✅ interface-contracts.md (always)                           │
│ ✅ world-design.md (when needed)                            │
│ ✅ Their own world code files (relevant ones only)          │
│ ❌ Combo state machine internals                            │
│ ❌ Ritual stacking math                                     │
│ ❌ Meta-progression save system                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 Sync Points — When Devs Must Coordinate

While daily work is independent, these moments require all 3 devs to sync:

| Sync Event | What Happens | Frequency |
|------------|-------------|-----------|
| **Interface Change** | If any dev needs to add/modify an interface, ALL 3 review and approve | As needed |
| **Integration Build** | Merge all branches, verify everything compiles and plays together | 2x per week |
| **Playtest Session** | All 3 play the game together, log feel issues | Weekly |
| **Sprint Review** | Demo what shipped, plan next sprint, update PKB | Bi-weekly |

### 5.4 Git Branch Strategy

```
main                              ← Always buildable, integration-tested
├── combat/combo-system           ← Dev 1's feature branches
├── combat/deflect-clash
├── roguelite/ritual-stacking     ← Dev 2's feature branches
├── roguelite/meta-progression
├── world/wave-manager            ← Dev 3's feature branches
├── world/enemy-ai-base
└── integration/sprint-3          ← Integration branch for testing before main merge
```

---

## Part 6: Task Breakdown — First 4 Weeks

### Week 1-2: Foundation (All 3 devs start simultaneously)

**Dev 1 Tasks (Combat):**
```
COMBAT-001: CharacterController2D (movement, jump, dash, gravity)
  → Agent context: project-brief + combat-design (movement section)
  → Template: combat-engineer

COMBAT-002: InputBufferSystem (queue inputs, buffer window)
  → Agent context: project-brief + combat-design (input section)
  → Template: combat-engineer

COMBAT-003: Basic Strike combo chain (3 hits + finisher, animation-driven)
  → Agent context: project-brief + combat-design + COMBAT-001 output
  → Template: combat-engineer

COMBAT-004: AttackData ScriptableObject (damage, knockback, launch, animation, timing)
  → Agent context: project-brief + interface-contracts (DamagePacket, AttackData)
  → Template: combat-engineer
```

**Dev 2 Tasks (Roguelite):**
```
RITUAL-001: RitualData ScriptableObject (family, category, trigger, effect, levels)
  → Agent context: project-brief + roguelite-design (ritual section)
  → Template: roguelite-architect

RITUAL-002: RitualFamily & RitualTrigger enums + shared data structures
  → Agent context: project-brief + interface-contracts (shared data section)
  → Template: roguelite-architect

RITUAL-003: CurrencyManager (crystals, imbued fruits, primordial seeds)
  → Agent context: project-brief + roguelite-design (meta-progression section)
  → Template: roguelite-architect

RITUAL-004: RitualStackCalculator (stacking math, ritual power, levels)
  → Agent context: project-brief + RITUAL-001 + roguelite-design (stacking rules)
  → Template: roguelite-architect + task-specific numeric examples
```

**Dev 3 Tasks (World):**
```
WORLD-001: WaveManager (spawn enemies, level bounds, camera stops)
  → Agent context: project-brief + world-design (wave section)
  → Template: world-builder

WORLD-002: EnemyBase class (implements IDamageable + IAttacker)
  → Agent context: project-brief + interface-contracts (IDamageable, IAttacker)
  → Template: world-builder

WORLD-003: Basic EnemyAI (idle, patrol, chase, attack, hit-react, death)
  → Agent context: project-brief + world-design (enemy section) + WORLD-002
  → Template: world-builder

WORLD-004: CameraController2D (follow player, level bounds, smooth leading)
  → Agent context: project-brief + world-design (camera section)
  → Template: world-builder
```

### Week 3-4: Core Systems

**Dev 1:** Deflect/Clash/Dodge system, Pressure/Stun meter, Hitbox manager
**Dev 2:** RitualSystem (trigger pipeline, active ritual management), Trinket system, Reward selection UI
**Dev 3:** Boss AI framework (phases, punish windows), Branching path system, HUD (health/mana bars, combo counter)

**End of Week 4 Integration Target:** A playable scene where one character can fight through one area with basic enemies, collect rituals, and face a simple boss. The full loop should work even if content is minimal.

---

## Part 7: Advanced Agent Pilot Patterns

### 7.1 Chain Agents for Complex Features

Some features span multiple files. Use **agent chaining** — the output of one agent becomes context for the next:

```
Agent A: "Design the data model for the Branching Path system"
  → Output: PathNodeData.cs, IslandData.cs, PathTypes.cs
  
Agent B: "Implement the path navigation logic using these data models"
  → Context includes: Agent A's output files
  → Output: PathNavigator.cs, PathSelectionUI.cs

Agent C: "Write tests for the path navigation logic"
  → Context includes: Agent B's output + interface contracts
  → Output: PathNavigatorTests.cs
```

### 7.2 The "Rubber Duck" Agent

When stuck on a design decision, spin up a lightweight agent with JUST the design question:

```markdown
# Agent: Game Design Consultant

## Context
[paste the specific design dilemma]

## Question
In Absolum, characters feel incomplete at the start of each run because
core abilities are locked behind Inspirations. Some players hate this.
For our game, should we:
A) Keep all core moves available from the start, make Inspirations purely additive
B) Lock some moves behind Inspirations like Absolum
C) Compromise: unlock core moves after first successful run, lock for subsequent difficulty tiers

Analyze pros/cons from a game design perspective.
```

No code output — just structured thinking. Cheap, fast, valuable.

### 7.3 The "Integration Validator" Agent

Run this AFTER merging branches:

```markdown
# Agent: Integration Validator

## Context
Here are the latest files from all 3 domains:
[paste ICombatEvents implementation]
[paste RitualSystem that subscribes to ICombatEvents]
[paste EnemyBase that implements IDamageable]

## Task
Verify that:
1. All interface methods are properly implemented (no NotImplementedException)
2. Event subscriptions match event signatures
3. No direct cross-domain references bypass the interfaces
4. ScriptableObject references won't cause null refs at runtime
Report any issues found.
```

### 7.4 Scaling Agents with the Team

As each dev gets comfortable, they'll develop their own agent refinements:
- Dev 1 might create a "Combo Feel Tuner" agent that takes recorded gameplay data and suggests timing adjustments
- Dev 2 might create a "Ritual Balance" agent that simulates build combinations and flags overpowered synergies
- Dev 3 might create an "Enemy Pattern Designer" agent that generates attack sequence definitions from natural language descriptions

These specialized agents become the team's **competitive advantage** — institutional knowledge encoded into reusable prompts.

---

## Part 8: Task Log Template

Every completed agent task gets logged. This is the team's learning engine:

```markdown
## [TASK-ID] — Task Name
**Date:** 2026-03-01
**Dev:** Dev 2
**Agent Template:** roguelite-architect
**Context Files Used:** project-brief.md, interface-contracts.md (IBuffProvider), 
  RitualData.cs, roguelite-design.md (stacking section only)
**Iterations:** 2
**Output Files:** RitualStackCalculator.cs
**Issues Encountered:** 
  - First pass: Agent made Ritual Power additive instead of multiplicative
  - Fix: Added explicit numeric example to task prompt:
    "Two +10% trinkets = 1.1 × 1.1 = 1.21, NOT 1.0 + 0.1 + 0.1 = 1.2"
**Lesson Learned:** Always include concrete numeric examples for math-heavy tasks.
  Agents interpret "multiplicative" correctly ~70% of the time without examples.
**Time Saved vs Manual:** ~45 min (would have taken ~2h manually, took ~1.25h with agent)
```

---

## Part 9: Quick Reference Card

Print this. Pin it to your monitor.

```
╔══════════════════════════════════════════════════════════════╗
║              AGENT PILOT — QUICK REFERENCE                  ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  BEFORE SPINNING UP AN AGENT, ASK:                          ║
║                                                              ║
║  1. Is my task atomic enough?                               ║
║     → If it touches 3+ files across domains, DECOMPOSE      ║
║                                                              ║
║  2. What's the MINIMUM context this agent needs?            ║
║     → project-brief.md (always)                             ║
║     → interface-contracts.md (always)                        ║
║     → Domain design spec (only relevant section)            ║
║     → Existing code (only files being modified/extended)    ║
║                                                              ║
║  3. Am I including code from OTHER domains?                 ║
║     → If yes, STOP. Use only the interfaces.                ║
║                                                              ║
║  4. Did I include a concrete example/test case?             ║
║     → Math tasks: include numeric example                   ║
║     → Logic tasks: include input→output example             ║
║     → UI tasks: include mockup or description               ║
║                                                              ║
║  5. How will I VALIDATE the output?                         ║
║     → Compiles?                                              ║
║     → Matches interface contract?                            ║
║     → Passes unit test?                                      ║
║     → Feels right in game? (combat tasks only)              ║
║                                                              ║
║  GOLDEN RULE: An agent that knows less, does better.        ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

---

## Appendix: Tool-Agnostic Execution

Agent Pilot is a **methodology**, not a specific tool. Each dev can use whichever LLM interface they prefer:

| Tool | Best For | How to Use with Agent Pilot |
|------|----------|----------------------------|
| **Claude Code** | Complex multi-file tasks, refactoring | Paste agent template as system prompt, add context files via CLAUDE.md or direct paste |
| **Claude.ai** (this) | Design discussions, code review, one-off scripts | Paste template + context in conversation, iterate |
| **Cursor** | In-editor coding tasks, smaller scope | Use .cursorrules for agent template, @-mention relevant files |
| **ChatGPT** | Alternative for agent execution | Custom GPT with template, paste context per task |
| **GitHub Copilot Agent** | CI/CD tasks, automated PRs | .github/agents/ with template, assign issues |

The point is: the **context assembly and task decomposition** is where the value lives, not the LLM tool. Switch tools freely. The methodology stays the same.
