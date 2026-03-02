# Tomato Fighters

**A 2D side-scrolling beat 'em up roguelite where 4 distinct character archetypes clash through stat-driven upgrade paths, defensive combat mastery, and elemental ritual stacking.**

## Vision

Tomato Fighters takes the defensive combat depth of Absolum (deflect, clash, punish) and layers on a **stat-driven character differentiation system** where each of the 4 characters starts with dramatically different base stats and evolves through a **Main + Secondary upgrade path** system. No two runs feel the same — even with the same character.

The game is built by a 3-developer team coordinated through **AgentPilot**, an AI-powered context orchestration engine that ensures each developer's domain stays cleanly separated while maintaining a shared knowledge base.

## Core Pillars

1. **Combat Depth Through Defense** — Deflect, Clash, and Punish are the primary skill expression. Players push INTO enemy attacks, not away.
2. **Stat-Driven Character Identity** — 4 archetypes (Tank, Melee DPS, Magician, Range) with 3 upgrade paths each. Main + Secondary path selection creates 6 builds per character.
3. **Roguelite Build-Crafting** — 8 elemental ritual families that stack multiplicatively. Path upgrades and rituals are independent systems that multiply together.

## Characters

| Character | Role | Signature | 3 Paths |
|-----------|------|-----------|---------|
| **Brutor** | Tank | 200 HP, 25 DEF, armored dash | Warden (aggro) / Bulwark (self-defense) / Guardian (team shields) |
| **Slasher** | Melee DPS | 2.0 ATK, 15% CRT, bloodlust stacking | Executioner (burst) / Reaper (AOE) / Shadow (evasion) |
| **Mystica** | Magician | 150 MNA, 8 MRG, arcane resonance | Sage (healer) / Enchanter (buffer) / Conjurer (summoner) |
| **Viper** | Range | 1.8 ranged ATK, distance bonus | Marksman (sniper) / Trapper (CC/harpoon) / Arcanist (mana charge) |

## Tech Stack

| Category | Technology |
|----------|-----------|
| Engine | Unity 2022 LTS (2D URP) |
| Language | C# |
| Data Architecture | ScriptableObjects |
| Input | Unity Input System (new) |
| Juice | DOTween |
| Co-op | Fishnet or Mirror (TBD) |
| Coordination | AgentPilot (context-optimized AI task execution) |

## Team

| Role | Domain | Pillar |
|------|--------|--------|
| **Dev 1** | Attack mechanics, character combat, path ability execution | Combat Sandbox |
| **Dev 2** | Buff/path system, rituals, meta-progression, stats | Roguelite Systems |
| **Dev 3** | Animation, movement, enemies, bosses, world, UI | World & Content |

## Repos

| Repo | Purpose |
|------|---------|
| `tomato-fighters/` | Unity code repository |
| `tomato-fighters-docs/` | Documentation, plans, task tracking (this repo) |

## Documentation

See [SUMMARY.md](./SUMMARY.md) for the full table of contents.

## Task Board

See [TASK_BOARD.md](./TASK_BOARD.md) for the complete task breakdown (60 tasks, 6 phases).

## Task Logbook

See [TASK_LOGBOOK.md](./TASK_LOGBOOK.md) for the running log of completed agent tasks, outcomes, and lessons learned.
