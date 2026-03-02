# TOMATO FIGHTERS — Character Archetypes & Upgrade Paths

**Version:** 1.0.0
**Game Type:** 2D Side-Scrolling Beat 'Em Up Roguelite
**Engine:** Unity 2022 LTS (2D URP)
**Based On:** Project Talamh framework (Absolum-inspired)

---

## Design Philosophy

### Stat-Driven Differentiation
Unlike Absolum where characters differ primarily by movesets and animations, **Tomato Fighters uses base stats as the primary differentiator from moment one.** Each character starts with a clear statistical identity that shapes how they play before any upgrades are acquired.

### The Main + Secondary Path System
Each character has **3 distinct upgrade paths**. During a run, a player commits to:
- **1 Main Path** — Gets full progression (Tiers 1-3), unlocking the path's signature ability at Tier 3
- **1 Secondary Path** — Gets partial progression (Tiers 1-2 only), providing supplementary tools
- **1 Locked Path** — Cannot be selected; that run, you sacrifice this path entirely

This creates **6 viable build combinations per character** (3 choose 2, ordered), encouraging replayability and team composition strategy in co-op.

### Path Selection Timing
- **Main Path** — Chosen at the **first upgrade shrine** (end of Area 1)
- **Secondary Path** — Chosen at the **second upgrade shrine** (end of Area 2)
- Once chosen, paths are locked for the run. This is a commitment, not a respec system.
- Rituals, trinkets, and inspirations from the roguelite layer stack ON TOP of the path upgrades.

### Upgrade Tiers
Each path has 3 tiers of power:

| Tier | Requirement | Effect |
|------|------------|--------|
| **Tier 1** | Select as Main or Secondary | Core mechanic unlocked |
| **Tier 2** | Defeat Area Boss (Main or Secondary) | Mechanic enhanced, new ability |
| **Tier 3** | Defeat Island Boss (Main ONLY) | Signature capstone ability |

---

## Shared Stat Framework

All characters share these 8 base stats. Starting values differ dramatically per archetype.

| Stat | Abbrev | What It Controls | Range |
|------|--------|-----------------|-------|
| **Health** | HP | Total hit points | 50-200 |
| **Defense** | DEF | Flat damage reduction per hit | 0-30 |
| **Attack** | ATK | Base physical damage multiplier | 0.5-2.0 |
| **Speed** | SPD | Movement speed + dash distance | 0.7-1.5 |
| **Mana** | MNA | Mana pool for Arcana + mana-based abilities | 30-150 |
| **Mana Regen** | MRG | Mana restored per second | 1-8 |
| **Crit Chance** | CRT | Chance for 1.5x damage | 0-25% |
| **Pressure Rate** | PRS | How fast this character fills enemy pressure meters | 0.5-2.0 |

### Stat Growth
Stats grow through:
1. **Path Upgrades** — Each tier gives specific stat boosts relevant to the path
2. **Rituals** — Elemental buffs that modify stats multiplicatively
3. **Trinkets** — Flat stat modifiers found during runs
4. **Soul Tree** — Permanent meta-progression (small but persistent)

---

## Character 1: BRUTOR (Tank)

### Identity
The immovable wall. Brutor exists to absorb punishment and control enemy attention. Low damage ceiling but functionally unkillable when played well. In co-op, Brutor decides who the enemies hit.

### Visual Concept
Massive tomato warrior, thick skin, small limbs relative to body. Carries a riot shield made from a pot lid. Wears kitchen apron as armor.

### Base Stats

| Stat | Value | Relative |
|------|-------|----------|
| HP | **200** | Highest |
| DEF | **25** | Highest |
| ATK | 0.7 | Low |
| SPD | 0.7 | Lowest |
| MNA | 50 | Low |
| MRG | 2 | Low |
| CRT | 5% | Low |
| PRS | 1.0 | Medium |

### Passive Ability: "Thick Skin"
Takes 15% less damage from all sources. Knockback distance reduced by 40%.

### Base Moveset (Before Paths)
- **Strike Chain:** 3-hit shield bash combo. Slow, short range, solid knockback.
- **Skill/Heavy:** Overhead shield slam. Hits grounded enemies (OTG capable).
- **Dash:** Short armored charge. Has super-armor during startup (can't be interrupted, still takes damage).
- **Deflect:** Standard timing window. On successful deflect, Brutor doesn't slide back at all.
- **Clash:** On successful clash, enemy is knocked back further than normal.

---

### Path A: WARDEN (Aggro)

**Theme:** Threat generation, taunt mechanics, damage-from-sustained-aggro. Brutor becomes a magnet that punishes enemies for ignoring him.

**Stat Bonuses:**
- Tier 1: +20 HP, +0.2 PRS
- Tier 2: +30 HP, +0.3 PRS, +0.1 ATK
- Tier 3: +40 HP, +0.5 PRS, +0.2 ATK

| Tier | Unlock | Description |
|------|--------|-------------|
| **T1: Provoke** | Active ability (cooldown: 8s) | Taunt all enemies within medium radius for 4 seconds. Taunted enemies MUST target Brutor. Taunted enemies deal 15% less damage. |
| **T2: Aggro Aura** | Passive | While 3+ enemies are targeting Brutor, he gains +25% ATK and +15% SPD. The more attention he gets, the more dangerous he becomes. |
| **T3: Wrath of the Warden** | Signature (Main only) | When Brutor takes cumulative damage equal to 50% of his max HP within 10 seconds, he enters **Wrath State** for 6 seconds: ATK doubled, attacks gain AOE shockwave, immune to stagger. Cooldown: 30s. |

**Playstyle:** Pull enemies, tank hits, ramp up damage through sustained aggro. In co-op, Brutor taunts enemies off the DPS characters and becomes a damage threat himself when focused.

---

### Path B: BULWARK (Singular Defense)

**Theme:** Personal invulnerability, counter-attacks, self-sustain. The solo tank that never dies.

**Stat Bonuses:**
- Tier 1: +30 HP, +5 DEF
- Tier 2: +40 HP, +8 DEF
- Tier 3: +50 HP, +12 DEF, +5% CRT

| Tier | Unlock | Description |
|------|--------|-------------|
| **T1: Iron Guard** | Active stance (toggle) | While holding block: movement halved, incoming damage reduced by 50%, successful deflects restore 5% HP. Can block Unstoppable (red) attacks but takes 30% damage instead of full. |
| **T2: Retaliation** | Passive (triggers on block) | Every 3rd hit blocked (including Iron Guard) triggers an automatic counter-strike that deals 150% ATK and staggers the attacker. Counter-strikes ignore enemy defense. |
| **T3: Fortress** | Signature (Main only) | **Unbreakable Stance:** Activate to become fully immune to damage and stagger for 3 seconds. During this window, ALL blocked damage is stored. When Fortress ends, release stored damage as a 360-degree shockwave. Max stored: 300% ATK. Cooldown: 45s. |

**Playstyle:** Stand your ground, block everything, counter-attack. Reward for perfect play is near-immortality. High skill ceiling in timing Iron Guard against Unstoppable attacks.

---

### Path C: GUARDIAN (Team Defense)

**Theme:** AOE protection, damage redirection, team healing. The co-op carry who makes everyone else harder to kill.

**Stat Bonuses:**
- Tier 1: +25 HP, +3 DEF, +20 MNA
- Tier 2: +35 HP, +5 DEF, +30 MNA
- Tier 3: +45 HP, +8 DEF, +40 MNA, +2 MRG

| Tier | Unlock | Description |
|------|--------|-------------|
| **T1: Shield Link** | Active ability (cooldown: 12s) | Create a protective link to the nearest ally. For 6 seconds, 30% of damage the ally takes is redirected to Brutor instead. Brutor's DEF applies to redirected damage. |
| **T2: Rallying Presence** | Passive aura (medium radius) | Allies near Brutor gain +10 DEF and regenerate 2% max HP per second. Brutor heals for 1% per ally in range. Stacks incentivize grouping. |
| **T3: Aegis Dome** | Signature (Main only) | Deploy a protective dome (large radius) for 5 seconds. Allies inside take 60% less damage. Enemies inside are slowed 30%. When the dome expires, it pulses healing: 20% max HP to all allies inside. Cooldown: 60s. |

**Playstyle:** Stay near allies, absorb and redirect damage, create safe zones. In solo play, Shield Link works on AI companions or summoned minions. The ultimate co-op tank.

---

### Brutor Build Combinations

| Main | Secondary | Identity | Playstyle Summary |
|------|-----------|----------|-------------------|
| Warden | Bulwark | **Berserker Tank** | Taunt + self-sustain. Solo viable aggro tank. |
| Warden | Guardian | **Raid Leader** | Taunt + team buffs. Co-op god, enemy magnet who heals allies. |
| Bulwark | Warden | **Counter-Fighter** | Personal defense + aggro ramp. Block → counter → wrath cycle. |
| Bulwark | Guardian | **Iron Medic** | Unkillable + team healing. Slow but teammates never die. |
| Guardian | Warden | **Protector** | Team shields + taunt. Pulls enemies then domes the team. |
| Guardian | Bulwark | **Bastion** | Team shields + personal blocks. Layers of defense on defense. |

---

## Character 2: SLASHER (Melee DPS)

### Identity
The glass cannon up close. Slasher lives in the danger zone — melee range — but outputs the highest raw damage. Rewards aggressive play and combo mastery. High skill floor, highest damage ceiling.

### Visual Concept
Lean, fast tomato with blade-like leaf appendages. Dual-wields kitchen knives. Wears a headband. Scarred from many fights.

### Base Stats

| Stat | Value | Relative |
|------|-------|----------|
| HP | 100 | Medium-Low |
| DEF | 8 | Lowest |
| ATK | **2.0** | **Highest** |
| SPD | **1.3** | High |
| MNA | 60 | Medium-Low |
| MRG | 3 | Medium |
| CRT | **15%** | High |
| PRS | **1.5** | High |

### Passive Ability: "Bloodlust"
Each consecutive hit within 3 seconds increases ATK by 3% (stacks up to 10x = +30%). Resets if Slasher doesn't hit anything for 3 seconds. Rewards non-stop aggression.

### Base Moveset (Before Paths)
- **Strike Chain:** 4-hit fast slash combo with a spinning finisher. Fast, tight hitboxes.
- **Skill/Heavy:** Lunging thrust. Covers distance. Causes wall-bounce on hit near walls.
- **Dash:** Fast forward dash. Can dash through enemies (not through attacks). Short i-frame window.
- **Deflect:** Tighter window than Brutor (less forgiving), but successful deflect grants guaranteed crit on next hit.
- **Clash:** On successful clash, Slasher immediately gets a free dash-cancel (combo extension).

---

### Path A: EXECUTIONER (Mono/Single-Target Damage)

**Theme:** Single-target burst, crit amplification, finisher damage. The boss-killer.

**Stat Bonuses:**
- Tier 1: +0.3 ATK, +5% CRT
- Tier 2: +0.4 ATK, +8% CRT
- Tier 3: +0.5 ATK, +12% CRT, +0.3 PRS

| Tier | Unlock | Description |
|------|--------|-------------|
| **T1: Mark for Death** | Active ability (cooldown: 10s) | Mark a single enemy for 8 seconds. Marked enemy takes 25% more damage from Slasher. Mark transfers to the nearest enemy if the marked target dies (remaining duration). |
| **T2: Execution Threshold** | Passive | Enemies below 30% HP take 50% more damage from Slasher. Crits against low-HP enemies deal 2.5x (up from 1.5x). Bosses at low HP melt. |
| **T3: Deathblow** | Signature (Main only) | Charged attack (1.5s windup, can be cancelled by taking damage). Deals 500% ATK to a single target. If the target is Marked AND below 30% HP, Deathblow deals 1000% ATK. Guaranteed crit. Cooldown: 25s. |

**Playstyle:** Identify the biggest threat, mark it, combo it down, execute with Deathblow. In boss fights, Executioner is the primary damage dealer once the boss enters low HP.

---

### Path B: REAPER (Multi-Target/AOE Damage)

**Theme:** Cleave, chain hits, crowd clearing. The wave specialist.

**Stat Bonuses:**
- Tier 1: +0.2 ATK, +15 HP
- Tier 2: +0.3 ATK, +20 HP, +0.1 SPD
- Tier 3: +0.4 ATK, +30 HP, +0.2 SPD

| Tier | Unlock | Description |
|------|--------|-------------|
| **T1: Cleaving Strikes** | Passive modifier | All melee attacks hit in a wider arc. Strike chain hits up to 3 enemies simultaneously (reduced damage: 100% first, 60% additional). Finishers always hit all enemies in range. |
| **T2: Chain Slash** | Active ability (cooldown: 6s) | Dash through up to 4 enemies in a line, dealing 80% ATK to each. If any enemy dies from Chain Slash, cooldown resets immediately. Enables "massacre chains" in dense waves. |
| **T3: Whirlwind** | Signature (Main only) | Spin attack lasting 4 seconds. Deals 60% ATK per hit to ALL enemies in melee radius, hitting every 0.3s. Slasher is immune to stagger during Whirlwind. Can move at 50% speed while spinning. Final hit launches all nearby enemies. Cooldown: 35s. |

**Playstyle:** Dive into groups, cleave everything, chain-dash through survivors. In co-op, Reaper handles wave clear while Executioner handles bosses.

---

### Path C: SHADOW (Dexterity/Evasion)

**Theme:** Dodge through enemies, dash attacks, i-frame extensions, hit-and-run. The untouchable.

**Stat Bonuses:**
- Tier 1: +0.2 SPD, +5% CRT
- Tier 2: +0.3 SPD, +8% CRT, +10 HP
- Tier 3: +0.4 SPD, +12% CRT, +20 HP

| Tier | Unlock | Description |
|------|--------|-------------|
| **T1: Phase Dash** | Passive modifier | Dash now has 2 charges (recharges every 3s each). Dashing through an enemy deals 40% ATK damage and applies a 2-second slow (30%). Slasher's i-frames extended by 50% during dash. |
| **T2: Afterimage** | Passive | After dashing, Slasher leaves an afterimage that persists for 1.5 seconds. If an enemy attacks the afterimage, Slasher appears behind that enemy and deals 120% ATK as a guaranteed crit (backstab). 4s internal cooldown per afterimage. |
| **T3: Thousand Cuts** | Signature (Main only) | Teleport between up to 6 enemies in rapid succession (0.2s each), dealing 150% ATK per hit. Each hit is a guaranteed crit. Slasher is invulnerable during the sequence. If fewer than 6 enemies exist, remaining hits target the last enemy. Cooldown: 40s. |

**Playstyle:** Never stand still. Dash through enemies, bait attacks into afterimages, backstab. Highest skill ceiling in the game. Misplayed Shadow dies instantly; mastered Shadow never gets hit.

---

### Slasher Build Combinations

| Main | Secondary | Identity | Playstyle Summary |
|------|-----------|----------|-------------------|
| Executioner | Reaper | **Assassin** | Mark priority target + cleave the rest. Versatile damage. |
| Executioner | Shadow | **Phantom Killer** | Mark → Phase Dash behind → backstab → Deathblow. Pure burst. |
| Reaper | Executioner | **Blender** | AOE clear + execute survivors. Nothing lives. |
| Reaper | Shadow | **Tornado** | Whirlwind in, dash out, afterimage bait. Safe AOE. |
| Shadow | Executioner | **Ghost Blade** | Untouchable assassin. Mark + Thousand Cuts = deletion. |
| Shadow | Reaper | **Duelist** | Phase through everything, cleave on pass-through. Stylish. |

---

## Character 3: MYSTICA (Magician/Support)

### Identity
The team enabler and the hardest character to master. Mystica has the lowest survivability but the highest impact on team composition. In solo, she's a glass cannon with minions. In co-op, she's the difference between a winning and losing run.

### Visual Concept
Ethereal tomato with glowing interior. Wears a wizard hat made from a leaf. Floats slightly off the ground. Carries a wooden spoon as a staff/wand.

### Base Stats

| Stat | Value | Relative |
|------|-------|----------|
| HP | **50** | **Lowest** |
| DEF | 5 | Very Low |
| ATK | 0.5 | Lowest |
| SPD | 1.0 | Medium |
| MNA | **150** | **Highest** |
| MRG | **8** | **Highest** |
| CRT | 5% | Low |
| PRS | 0.5 | Lowest |

### Passive Ability: "Arcane Resonance"
When Mystica casts any ability, all nearby allies gain +5% damage for 3 seconds (stacks up to 3x = +15%). Encourages constant casting.

### Base Moveset (Before Paths)
- **Strike Chain:** 3-hit magical projectile burst (short range, piercing). Low damage but safe.
- **Skill/Heavy:** Arcane bolt — medium range, homing, moderate damage. Primary ranged poke.
- **Dash:** Blink teleport — short range but instant, excellent i-frames. Goes through everything.
- **Deflect:** Mystica's deflect creates a brief magic barrier. Same window, but on success also restores 5% mana.
- **Clash:** On successful clash, Mystica's next ability costs no mana.

---

### Path A: SAGE (Healer)

**Theme:** Sustain healing, burst healing, revival. The lifeline.

**Stat Bonuses:**
- Tier 1: +30 HP, +20 MNA, +2 MRG
- Tier 2: +40 HP, +30 MNA, +3 MRG
- Tier 3: +50 HP, +40 MNA, +4 MRG

| Tier | Unlock | Description |
|------|--------|-------------|
| **T1: Mending Aura** | Toggle (drains 3 MNA/s) | While active, all allies within medium radius regenerate 3% max HP per second. Mystica can still attack and cast while Mending Aura is active. |
| **T2: Purifying Burst** | Active ability (cooldown: 15s, costs 40 MNA) | Instantly heal all allies within large radius for 25% their max HP. Remove all negative status effects (burn, slow, poison, stun). The burst also damages undead/cursed enemies for 100% ATK. |
| **T3: Resurrection** | Signature (Main only) | When an ally is downed (HP reaches 0), Mystica can channel for 3 seconds to revive them at 50% HP. Mystica is vulnerable during channeling but gains 50% damage reduction. If interrupted, ability goes on half cooldown (30s). Full cooldown: 60s. Only works once per ally per run (can revive each ally once). |

**Playstyle:** Stay at safe distance, keep Mending Aura running, save Purifying Burst for emergencies, hold Resurrection for the clutch moment. Mana management is critical — running dry means the team loses its safety net.

---

### Path B: ENCHANTER (Buffer)

**Theme:** Stat buffs, elemental infusion, aura amplification. Makes the whole team stronger.

**Stat Bonuses:**
- Tier 1: +20 MNA, +2 MRG, +0.1 SPD
- Tier 2: +30 MNA, +3 MRG, +0.1 SPD, +15 HP
- Tier 3: +40 MNA, +4 MRG, +0.2 SPD, +25 HP

| Tier | Unlock | Description |
|------|--------|-------------|
| **T1: Empower** | Active ability (cooldown: 10s, costs 25 MNA) | Target an ally (or self). For 8 seconds, target gains +30% ATK and +20% SPD. In solo play, can target AI companions. |
| **T2: Elemental Infusion** | Active ability (cooldown: 14s, costs 35 MNA) | Infuse an ally's attacks with the dominant elemental ritual family in the party's build. Infused attacks trigger that element's effects for 10 seconds (e.g., Fire infusion = attacks apply burn). If no rituals active, defaults to bonus Physical damage (+20%). |
| **T3: Arcane Overdrive** | Signature (Main only) | For 8 seconds, ALL active buffs on ALL allies are doubled in effect (Empower becomes +60% ATK, +40% SPD; Mending Aura heals 6%/s; Elemental Infusion procs twice). Additionally, all allies' cooldowns tick 50% faster. Mystica's mana drains rapidly (10 MNA/s). Cooldown: 50s. |

**Playstyle:** Buff rotation — Empower the DPS, Infuse the Tank, maintain uptime. Enchanter Mystica is a "buff bot" that transforms mediocre allies into powerhouses. Requires game knowledge to know WHEN to buff WHO.

---

### Path C: CONJURER (Summoner)

**Theme:** Summon minions, deploy totems, construct turrets. The army builder.

**Stat Bonuses:**
- Tier 1: +20 HP, +20 MNA, +0.1 ATK
- Tier 2: +30 HP, +30 MNA, +0.2 ATK
- Tier 3: +40 HP, +40 MNA, +0.3 ATK, +2 MRG

| Tier | Unlock | Description |
|------|--------|-------------|
| **T1: Summon Sproutling** | Active ability (cooldown: 12s, costs 30 MNA) | Summon a small plant minion (HP: 40, ATK: 0.6, lasts 20s or until killed). Max 2 active. Sproutlings auto-attack the nearest enemy. They inherit 30% of Mystica's ritual effects. |
| **T2: Deploy Totem** | Active ability (cooldown: 18s, costs 45 MNA) | Place a stationary totem (HP: 80, lasts 15s). Totem pulses every 2 seconds: damages enemies in radius (50% ATK) and slows them 20%. Max 1 active. Totem inherits Mystica's elemental ritual family (Fire totem burns, Lightning totem chains, etc.). |
| **T3: Summon Golem** | Signature (Main only) | Summon a massive Golem (HP: 300, ATK: 1.5, DEF: 15, lasts 20s). Golem has its own AI: taunts enemies, slams for AOE damage, and absorbs hits for Mystica. Golem inherits 50% of Mystica's ritual effects. While Golem is alive, Sproutling max +1 (3 total). Cooldown: 60s. |

**Playstyle:** Summon army, position totems, hide behind Golem. Solo Mystica's best path — minions compensate for her terrible HP. Manages a small economy of summon timers and mana.

---

### Mystica Build Combinations

| Main | Secondary | Identity | Playstyle Summary |
|------|-----------|----------|-------------------|
| Sage | Enchanter | **Holy Priestess** | Heal + buff. The ultimate co-op support. |
| Sage | Conjurer | **Spirit Healer** | Heal team + summon bodyguards. Solo-viable healer. |
| Enchanter | Sage | **War Chanter** | Buff first, heal second. Offensive support. |
| Enchanter | Conjurer | **Puppet Master** | Buff summons + Empower allies. Army of boosted minions. |
| Conjurer | Sage | **Necromancer** | Summon army + sustain them and self. Solo specialist. |
| Conjurer | Enchanter | **Overlord** | Summon army + buff them. Empowered Golem is terrifying. |

---

## Character 4: VIPER (Range Attacker)

### Identity
Zone controller and crowd manager. Viper fights from distance, controlling the battlefield through damage, crowd control, or mana-powered devastation. Safe positioning, punishing to master against fast-closing enemies.

### Visual Concept
Sleek, serpentine tomato with elongated form. Uses a slingshot (or crossbow made from toothpicks) as primary weapon. Wears goggles. Has a belt of ammunition pouches.

### Base Stats

| Stat | Value | Relative |
|------|-------|----------|
| HP | 80 | Low |
| DEF | 10 | Low |
| ATK | 0.6 (melee) / **1.8 (ranged)** | Split scaling |
| SPD | 1.1 | Medium-High |
| MNA | **120** | High |
| MRG | **6** | High |
| CRT | 10% | Medium |
| PRS | 0.8 | Low-Medium |

### Passive Ability: "Distance Bonus"
Ranged attacks deal +2% damage per unit of distance from the target (max +30% at max range). Encourages maintaining distance. Melee attacks still work but at reduced ATK (0.6).

### Base Moveset (Before Paths)
- **Strike Chain:** 3-shot burst. Fast projectiles, medium range. Each shot has slight tracking.
- **Skill/Heavy:** Charged shot — hold to charge (up to 1.5s), deals 150%-300% ranged ATK. Pierces first enemy hit.
- **Dash:** Backflip dash — moves Viper backwards and fires a quick shot during the flip. Built-in attack + retreat.
- **Deflect:** Viper's deflect fires a projectile back at the attacker (reflect). Same timing window.
- **Clash:** On successful clash, Viper drops a small caltrops area at her feet (slows enemies for 3s).

---

### Path A: MARKSMAN (Pure Ranged Damage)

**Theme:** Raw ranged DPS, projectile enhancement, crit stacking. The sniper.

**Stat Bonuses:**
- Tier 1: +0.3 ranged ATK, +5% CRT
- Tier 2: +0.4 ranged ATK, +8% CRT, +10 HP
- Tier 3: +0.5 ranged ATK, +12% CRT, +0.1 SPD

| Tier | Unlock | Description |
|------|--------|-------------|
| **T1: Piercing Shots** | Passive | All ranged attacks pierce through enemies (hit everything in line). Damage reduces 20% per enemy pierced. Charged Shot pierces at full damage for the first 2 enemies. |
| **T2: Rapid Fire** | Active ability (cooldown: 8s) | Fire 8 shots in 2 seconds (4x normal fire rate). Each shot deals 60% ranged ATK. Combined with Piercing, this shreds packed groups. Can move at 70% speed during Rapid Fire. |
| **T3: Killshot** | Signature (Main only) | Ultra-charged sniper shot (2s charge, Viper cannot move during charge). Deals 800% ranged ATK. Guaranteed crit. Ignores DEF. If Killshot kills the target, the projectile continues through ALL enemies for 400% each. Cooldown: 30s. |

**Playstyle:** Maintain maximum distance, use Piercing to damage lines, Rapid Fire for burst, Killshot for eliminations. Pure DPS from safe range. Vulnerable if enemies close the gap.

---

### Path B: TRAPPER (Harpoon/Crowd Control)

**Theme:** Stun, immobilize, pull, area denial. The controller.

**Stat Bonuses:**
- Tier 1: +20 HP, +0.1 SPD, +0.2 PRS
- Tier 2: +30 HP, +0.2 SPD, +0.3 PRS
- Tier 3: +40 HP, +0.2 SPD, +0.5 PRS, +15 MNA

| Tier | Unlock | Description |
|------|--------|-------------|
| **T1: Harpoon Shot** | Active ability (cooldown: 6s, costs 15 MNA) | Fire a harpoon that travels full screen. First enemy hit is **immobilized** for 2 seconds (cannot move or attack, but can be damaged). Harpoon deals 70% ranged ATK + fills 20% of target's pressure meter. |
| **T2: Trap Net** | Active ability (cooldown: 12s, costs 25 MNA) | Deploy a trap on the ground. First enemy to walk over it is snared for 3 seconds + takes 100% ranged ATK damage. Trap lasts 10 seconds if not triggered. Max 2 traps active. Traps are invisible to enemies. |
| **T3: Anchor Chain** | Signature (Main only) | Fire a massive anchor that chains to the ground on impact. All enemies within large radius are **pulled toward the anchor** and slowed 50% for 4 seconds. Chained enemies take 30% more damage from all sources. Anchor persists for 6 seconds, re-pulling enemies that try to leave. Cooldown: 40s. |

**Playstyle:** Control enemy movement — Harpoon priority targets, trap chokepoints, Anchor Chain for team combos. In co-op, Trapper sets up the kills; Slasher finishes them. Best pressure builder in the game.

---

### Path C: ARCANIST (Mana Charge/Mana Attacks)

**Theme:** Mana as a resource AND a weapon. Charge mana for devastating magical attacks. High risk/reward mana management.

**Stat Bonuses:**
- Tier 1: +30 MNA, +3 MRG
- Tier 2: +40 MNA, +4 MRG, +0.2 ranged ATK
- Tier 3: +50 MNA, +5 MRG, +0.3 ranged ATK, +5% CRT

| Tier | Unlock | Description |
|------|--------|-------------|
| **T1: Mana Charge** | Active (channeled) | Hold ability button to charge mana. Charging fills a **Mana Charge meter** (0-100%). While charging, Viper is stationary but gains 30% damage reduction. Mana Charge decays 10%/second when not channeling. Charged mana amplifies the next ability. |
| **T2: Mana Blast** | Active ability (costs current Mana Charge) | Consumes all Mana Charge to fire a beam. Damage scales with charge: 25% charge = 100% ATK, 50% = 250% ATK, 75% = 400% ATK, 100% = 600% ATK. Beam is wide and pierces all enemies. At 100% charge, beam also stuns for 1.5s. No cooldown — limited by charge time. |
| **T3: Mana Overload** | Signature (Main only) | Instantly fill Mana Charge to 100% AND gain **Overcharged** state for 10 seconds. While Overcharged: all abilities cost double mana but deal 2x damage, Mana Blast can be fired at 150% charge (900% ATK, 2.5s stun, pulls enemies toward beam center). When Overcharged ends, Viper is mana-drained (0 MNA) and cannot regen for 3 seconds. Cooldown: 50s. |

**Playstyle:** Rhythm of charge → blast → charge → blast. Arcanist has the highest single-attack damage ceiling in the game (Mana Overload → 150% Mana Blast = 900% ATK) but punishes poor timing with the post-Overload vulnerability window.

---

### Viper Build Combinations

| Main | Secondary | Identity | Playstyle Summary |
|------|-----------|----------|-------------------|
| Marksman | Trapper | **Sharpshooter** | Harpoon → Piercing Rapid Fire → Killshot combo. |
| Marksman | Arcanist | **Artillery** | Charge between Rapid Fire bursts. Killshot + Mana Blast delete bosses. |
| Trapper | Marksman | **Warden** | Lock down + DPS. Anchor Chain → Piercing Rapid Fire devastation. |
| Trapper | Arcanist | **Lockdown Mage** | Immobilize → Charge → Mana Blast on frozen targets. Safe combo. |
| Arcanist | Marksman | **Siege Cannon** | Overcharged Mana Blasts + Piercing Shots for sustained damage. |
| Arcanist | Trapper | **Battle Mage** | Harpoon → Charge → Mana Blast on stunned target. CC + Burst. |

---

## Team Composition Matrix

### Recommended 2-Player Co-op Pairs

| Pair | Why It Works |
|------|-------------|
| **Brutor (Guardian) + Slasher (Executioner)** | Tank absorbs hits, DPS nukes bosses. Classic. |
| **Brutor (Warden) + Viper (Trapper)** | Double CC — taunted AND harpoon'd enemies. Total control. |
| **Slasher (Reaper) + Mystica (Enchanter)** | Buffed Whirlwind Slasher = screen clear. Glass cannon + buff bot. |
| **Viper (Arcanist) + Mystica (Sage)** | Healer keeps fragile Viper alive for Overcharged Mana Blasts. |
| **Mystica (Conjurer) + Viper (Marksman)** | Minion frontline + ranged DPS backline. Neither gets touched. |
| **Brutor (Bulwark) + Mystica (Enchanter)** | Unkillable tank + damage buffs = slow but inevitable victory. |

### Full 4-Player Dream Teams

| Team Comp | Roles | Strategy |
|-----------|-------|----------|
| **Brutor + Slasher + Mystica + Viper** | Tank/DPS/Support/Control | Standard balanced team. Everyone has a role. |
| **2x Slasher + Mystica + Viper** | DPS/DPS/Buff/CC | Glass cannon rush. Enchanter buffs both Slashers. High risk. |
| **Brutor + 2x Viper + Mystica** | Tank/Range/Range/Heal | Siege comp. Tank holds line, Vipers bombard from range. |

---

## Stat Comparison At A Glance

| Stat | Brutor | Slasher | Mystica | Viper |
|------|--------|---------|---------|-------|
| HP | **200** | 100 | 50 | 80 |
| DEF | **25** | 8 | 5 | 10 |
| ATK | 0.7 | **2.0** | 0.5 | 0.6m / 1.8r |
| SPD | 0.7 | **1.3** | 1.0 | 1.1 |
| MNA | 50 | 60 | **150** | 120 |
| MRG | 2 | 3 | **8** | 6 |
| CRT | 5% | **15%** | 5% | 10% |
| PRS | 1.0 | **1.5** | 0.5 | 0.8 |

---

## Integration with Roguelite Layer

### How Paths Interact with Rituals
Path upgrades and rituals are **independent systems that multiply together**:

- **Brutor (Warden)** + Fire rituals = Taunted enemies burn when attacking Brutor
- **Slasher (Executioner)** + Lightning rituals = Marked enemy chains lightning to nearby foes when hit
- **Mystica (Conjurer)** + Necro rituals = Summons heal Mystica on kills
- **Viper (Arcanist)** + Cosmic rituals = Mana Blast triggers cosmic AoE on impact

The `IBuffProvider` interface handles this — path upgrades are stat modifications, rituals add on-hit/on-trigger effects through the same combat event pipeline defined in `ICombatEvents`.

### Character-Specific Inspirations
Each character has **6 Inspirations** (2 per path, unlocked during runs from minibosses):

| Character | Path | Inspiration 1 | Inspiration 2 |
|-----------|------|---------------|---------------|
| **Brutor** | Warden | Provoke also damages taunted enemies | Wrath duration +3s |
| | Bulwark | Iron Guard reflects 20% blocked damage | Counter-strike hits all nearby enemies |
| | Guardian | Shield Link chains to 2nd ally | Aegis Dome also cleanses debuffs |
| **Slasher** | Executioner | Mark spreads to nearby enemies on kill | Deathblow has no windup below 10% HP |
| | Reaper | Chain Slash hits 6 enemies (up from 4) | Whirlwind pulls enemies inward |
| | Shadow | Phase Dash has 3 charges (up from 2) | Afterimage persists 3s and can trigger twice |
| **Mystica** | Sage | Mending Aura costs 50% less mana | Resurrection heals for 75% (up from 50%) |
| | Enchanter | Empower affects 2 targets | Elemental Infusion lasts 15s (up from 10s) |
| | Conjurer | Sproutling max +1 (3/4 total) | Golem gains a ranged attack |
| **Viper** | Marksman | Piercing at full damage for 3 enemies | Killshot cooldown -10s |
| | Trapper | Harpoon immobilize +1s | Anchor Chain radius +40% |
| | Arcanist | Mana Charge decays 50% slower | Overcharged state +5s duration |

---

## Implementation Notes (For Unity/Agent Pilot)

### ScriptableObject Architecture
```
ScriptableObjects/
├── Characters/
│   ├── BrutorBaseStats.asset
│   ├── SlasherBaseStats.asset
│   ├── MysticaBaseStats.asset
│   └── ViperBaseStats.asset
├── Paths/
│   ├── Brutor/
│   │   ├── WardenPath.asset
│   │   ├── BulwarkPath.asset
│   │   └── GuardianPath.asset
│   ├── Slasher/
│   │   ├── ExecutionerPath.asset
│   │   ├── ReaperPath.asset
│   │   └── ShadowPath.asset
│   ├── Mystica/
│   │   ├── SagePath.asset
│   │   ├── EnchanterPath.asset
│   │   └── ConjurerPath.asset
│   └── Viper/
│       ├── MarksmanPath.asset
│       ├── TrapperPath.asset
│       └── ArcanistPath.asset
└── Inspirations/
    ├── Brutor/  (6 assets)
    ├── Slasher/ (6 assets)
    ├── Mystica/ (6 assets)
    └── Viper/   (6 assets)
```

### Key Interfaces to Extend
The existing Talamh interface contracts need these additions for the path system:

```csharp
// New interface for Path System
public interface IPathProvider
{
    PathData MainPath { get; }
    PathData SecondaryPath { get; }
    int MainPathTier { get; }    // 1-3
    int SecondaryPathTier { get; } // 1-2
    bool HasPath(PathType type);
    float GetPathStatBonus(StatType stat);
}

// Extend IBuffProvider
public interface IBuffProvider
{
    // ... existing methods ...
    float GetPathDamageMultiplier();
    float GetPathDefenseMultiplier();
    float GetPathSpeedMultiplier();
    List<PathAbility> GetActivePathAbilities();
}

// Path-specific enums
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
    Health, Defense, Attack, Speed, Mana, ManaRegen, CritChance, PressureRate
}
```

### Developer Ownership Mapping

| System | Owner | Pillar |
|--------|-------|--------|
| Base stats + stat growth | Dev 2 (Roguelite) | Pillar 2 |
| Path selection UI | Dev 3 (World) | Pillar 3 |
| Path ability execution | Dev 1 (Combat) | Pillar 1 |
| Path data (ScriptableObjects) | Dev 2 (Roguelite) | Pillar 2 |
| Character movesets + animation | Dev 1 (Combat) | Pillar 1 |
| Character selection (hub) | Dev 2 (Roguelite) | Pillar 2 |
| Inspiration drops from minibosses | Dev 3 (World) | Pillar 3 |
| Buff calculation (path + ritual + trinket) | Dev 2 (Roguelite) | Pillar 2 |
