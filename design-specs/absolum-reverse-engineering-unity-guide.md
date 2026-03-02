# Reverse-Engineering Absolum: Building a Beat 'Em Up Roguelite in Unity

## Game Overview

**Absolum** is a 2D side-scrolling beat 'em up fused with roguelite mechanics, developed by Guard Crush Games (Streets of Rage 4) with animation by Supamonks, published by Dotemu. Released October 9, 2025. 94% positive on Steam, 87 Metacritic.

**Genre DNA:** Beat 'em up (Golden Axe, Streets of Rage) + Roguelite (Hades, Dead Cells) + Action RPG elements

**Core Fantasy:** Classic arcade brawler with modern roguelite progression — every run feels different through stacking passive abilities, branching paths, and build-crafting.

---

## 1. COMBAT SYSTEM (The Heart of the Game)

### 1.1 Input Actions
The game uses **3 primary attack buttons + defensive inputs:**

| Input | Name | Function |
|-------|------|----------|
| X | **Strike** (Light) | Fast basic attack, combo starter |
| Y | **Skill** (Heavy) | Slower, more powerful, has a parry/clash window |
| B | **Arcana** (Special) | Mana-consuming special ability |
| Dash | **Dash/Dodge** | Directional dodge with deflect window |
| Jump | **Jump** | Aerial attacks, juggle extensions |

### 1.2 Combo System Architecture

**Key Insight:** Combos are NOT just "mash X." The depth comes from the interplay between offensive and defensive mechanics.

**Basic Combo Chain:** `Strike → Strike → Strike → Finisher`
- This is the *simplest* path, NOT the optimal one
- The finisher auto-plays at end of Strike chain

**Advanced Combo Routes:**
- **Dash Cancelling:** Most attacks are cancellable on-hit into Dash or Run. This is the primary combo extension tool.
- **Dash Jump:** Forward dash can be immediately cancelled into jump, creating a pseudo "jump cancel"
- **Wall Bounces:** Certain moves cause wall bounces. These are **unlimited** and each does minor damage without building extra pressure — use as many as possible in a combo
- **Air Juggles:** Launching enemies into the air allows extended combos. Gale element keeps enemies airborne longer.

**Example Advanced Galandra Combo:**
```
X-X-X-Y-B → Jump → JumpX → JumpX → X-X-X-(Finisher)
```
- Cancel Y (Skill) with B (Arcana)
- Miss a clash attempt → follow with dash to catch a deflect

### 1.3 Defensive Mechanics (Where the Real Depth Lives)

This is what separates Absolum from basic brawlers. The devs intentionally moved depth INTO the defensive systems:

| Mechanic | Input | Reward |
|----------|-------|--------|
| **Dodge** | Dash button + direction | i-frames during dodge, safe repositioning |
| **Deflect** | Dash with correct timing on incoming attack | Generous window, turns hits into punish opportunities |
| **Clash** | Heavy attack (Y/Skill) timed against enemy attack | Tighter window than deflect, but MORE rewarding |
| **Punish** | Hit boss after their big attack whiffs/is deflected | Red hit spark indicator, deals massively increased damage |

**Overpressure/Stun System:**
- Repeatedly punishing + overwhelming with regular attacks builds a hidden "pressure" meter on enemies
- When full → enemy is **stunned** and can be juggled freely
- After a few seconds of juggle → enemy enters **invincible state** (blinks white, screen closes up briefly) → recovers and resumes attack rotation

### 1.4 Tech Hit vs OTG (On The Ground)
- **Tech Hit:** Hitting downed enemies — they CAN recover (land on feet)
- **OTG:** Arcana moves hit downed enemies but DON'T count as Tech Hits — enemies can't recover from repeated Arcana hits on the ground
- This distinction is critical for combo design

### 1.5 Anti-Spam: Repetitive Penalty
- A "Repetitive" debuff kicks in when you spam the same move
- Reduces damage output
- Forces players to use varied combos and the full toolkit

### 1.6 Visual Attack Signals
- Enemy attacks have **color-coded visual cues:**
  - Normal attacks: Can be deflected/clashed
  - **Red signal attacks:** UNSTOPPABLE — must dodge, cannot deflect or clash

---

## 2. CHARACTER SYSTEM

### 2.1 Four Playable Characters (Archetypes)

| Character | Archetype | Weapon | Playstyle |
|-----------|-----------|--------|-----------|
| **Galandra** | All-rounder Swordfighter | Colossal sword + Necromancy | Balanced, strong punish windows, readable animations |
| **Karl** | Tanky Brawler | Blunderbuss + Fists | Close-range + explosive ranged pokes, compensates for melee range |
| **Cider** | Agile Skirmisher | Clockwork prosthetic parts | High mobility, acrobatic combos, mechanical gadgets |
| **Brome** | Ranged Wizard | Magic (frog mage) | Crowd control, ranged setup, elemental focus |

### 2.2 Character Progression Layers

Each character has a **3-layer progression system:**

1. **Base Layer** — Innate abilities, basic moveset (stripped down at start)
2. **Run Layer** — Rituals, Trinkets, Inspirations acquired during current run (LOST on death)
3. **Meta Layer** — Permanent upgrades via Soul Tree, unlocked Arcanas, unlocked Inspirations

**Design Controversy:** Core abilities like Galandra's dive kick and 3-hit sword combo are locked behind "Inspirations" found during runs — characters feel incomplete at baseline. This is intentional to create progression feel but divisive among players.

### 2.3 Arcana (Super Abilities)
- Each character has multiple **Arcana** options (mana-consuming specials)
- Selected before each run
- One Arcana glows = bonus XP for picking it
- Second Arcana unlocked mid-run after completing first island
- Unlocked permanently via **Primordial Seeds** currency

### 2.4 Inspirations
- Character-specific mechanical tweaks (new moves, passive abilities)
- Dropped from **minibosses** during runs
- Examples: Galandra's Dive Kick, multi-hit Air Spinning Blade
- Some are permanently unlockable, some are run-specific finds

---

## 3. ROGUELITE SYSTEMS

### 3.1 Rituals (Core Build-Crafting — Like Hades Boons)

Rituals are the **primary run customization mechanic.** They are passive/active modifiers that change how you deal damage, resist, or manipulate combat.

**8 Elemental Families:**
- **Fire** — Burn damage, Blazing Dash
- **Lightning** — Chain Lightning, Lightning Strike
- **Water** — Tidal Waves (Wave Dash, Wave Landing)
- **Thorn** — Bramble Knives spawned on finishers/dodges (carry up to 20)
- **Gale/Wind** — Keeps enemies airborne longer for extended juggles
- **Time** — Summon echoes that copy your Arcana attacks
- **Cosmic** — Can break the game — infinite ultimates, AoE attacks
- *(+more unlocked via progression)*

**Ritual Categories:**
| Category | Description |
|----------|-------------|
| **Core** | Main mechanic of a family (Burn, Chain Lightning, etc.) — appended to attacks |
| **General** | Global upgrades themed to family but independent of loadout |
| **Enhancement** | Amplify existing ritual effects |
| **Twin** | Combines TWO families (e.g., Lightning + Fire). Requires having rituals from both families |

**Ritual Progression:**
- Levels 1-3 (Level 2 ≈ 50% stronger, Level 3 ≈ 100% stronger)
- **Stacking:** Multiple rituals enhancing same mechanic increase each other's power
- **Ritual Power:** Hidden stat, increased via Trinkets (Moxes, Crown of Harmony). Multiplicative scaling.
- Up to **3 different families per run** naturally, but can add extras by purchasing off-pool rituals

**Acquisition:**
- Offered at end of each cleared non-boss area (pick 1 of 2, or 3 with upgrade)
- Also from: Fallen Hunters (dead exiled enemies, costs HP), certain shops, NPCs during runs
- ~40 total rituals to unlock via hub NPCs using rare currencies

### 3.2 Trinkets
- Stat-boosting items found during runs
- Buff specific stats or add conditional effects
- Can boost Ritual Power, grant bonuses for perfect dodges, etc.
- Purchasable at shops

### 3.3 Build Synergy Philosophy
The game rewards **coherent elemental/mechanical builds:**
- Match damage type (physical → physical-boosting rituals)
- Don't mix opposing effects (bleed sustained damage + burst immediate damage = diluted)
- Chain trigger conditions (e.g., "on perfect deflect" → "spawn knives" → "knives apply burn")
- Example: Brome with echo-spawning Time rituals + Proton Cannon Arcana = double special attacks
- Example: Galandra with Gale element + Dive Kick + Air Spin = infinite air juggles

---

## 4. META-PROGRESSION

### 4.1 Three Currencies

| Currency | Source | Use |
|----------|--------|-----|
| **Crystals** | Carry over between runs, from combat/breakables | Soul Tree upgrades, reroll ritual offerings |
| **Imbued Fruits** | Rare, from completing tasks | Upgrade Rituals (make them stronger permanently) |
| **Primordial Seeds** | Rare | Unlock new Arcanas and Inspirations |

### 4.2 Soul Tree (Hub Upgrade Tree)
- Permanent passive upgrades: increased health, damage, self-revives, rare item chances
- Not a branching skill tree — more of a list you unlock down
- Modular and fluid — not rigid RPG tree
- Some upgrades allow Level 2/3 Rituals to spawn directly without upgrading

### 4.3 Hub Area ("The Hearth")
- Permanently located on first island
- Contains:
  - **Vikhana** — NPC that unlocks Twin Rituals
  - **Ederig** — NPC that unlocks Inspirations
  - Shops for pre-equipping trinkets
  - Soul Tree access
  - Character/Arcana selection
  - Quest tracking

---

## 5. WORLD & RUN STRUCTURE

### 5.1 Run Layout
- Full run = **3 islands** traversed sequentially
- **4 total islands** exist — path choices determine which 3 you visit
- Each island has: unique enemies, sub-bosses, environment, culture, NPCs
- **Boss at end of each island**

### 5.2 Map & Branching Paths
- **NOT procedurally generated levels** — maps are hand-crafted and fixed
- Branching PATHS through the map — choose routes at forks
- Different routes offer: shops, challenge rooms, Inspirations, mercenaries/mounts, quests
- Some enemy types are randomized per map section
- Different bosses depending on direction chosen

### 5.3 Between-Area Rewards
- After clearing a combat area: pick a Ritual, gold, or crystals
- Optional paths with tougher minibosses = better rewards
- Breakable objects (barrels, crates) contain: food (health), gold, crystals

### 5.4 Mounts & Companions
- Rideable mounts found in certain areas
- AI mercenaries can be hired
- Even chickens can fight alongside you
- Mount-related rituals exist but mounts are rare, making those rituals feel like wasted picks

### 5.5 Quests & World Evolution
- Side quests open new routes and NPC characters
- Quest markers on map once picked up
- World changes based on choices — new pathways, story beats shift
- Defeating "final" boss opens new quest types and difficulty mode
- Talk to everyone with speech bubbles

---

## 6. CO-OP SYSTEM

- **2-player only** (local couch + online with rollback netcode)
- Combine elemental powers for chained combos
- Players do NOT share gold
- No trading system
- Online randoms can buy shop items from under you
- No player scaling for matchmaking
- Can have 4 heroes on screen with AI companions, but only 2 human players

---

## 7. UNITY IMPLEMENTATION BLUEPRINT

### 7.1 Project Architecture

```
Assets/
├── Scripts/
│   ├── Combat/
│   │   ├── ComboSystem.cs          // Input buffer, combo state machine
│   │   ├── AttackData.cs           // ScriptableObject: damage, knockback, launch, animation
│   │   ├── HitboxManager.cs        // Attack hitbox activation/deactivation
│   │   ├── DefenseSystem.cs        // Deflect, Clash, Dodge, i-frames
│   │   ├── DamageSystem.cs         // Health, damage calculation, Punish multiplier
│   │   ├── PressureSystem.cs       // Overpressure/stun meter
│   │   ├── RepetitiveTracker.cs    // Anti-spam damage reduction
│   │   └── WallBounce.cs           // Wall collision → bounce physics
│   ├── Character/
│   │   ├── CharacterController2D.cs // Movement, jump, dash
│   │   ├── CharacterAnimator.cs     // Animation state management
│   │   ├── ArcanaSystem.cs          // Special ability with mana cost
│   │   ├── InspirationSlots.cs      // Runtime ability unlocks
│   │   └── CharacterData.cs         // SO: base stats per character
│   ├── Roguelite/
│   │   ├── RitualSystem.cs          // Ritual stacking, family management
│   │   ├── RitualData.cs            // SO: ritual definitions, levels, families
│   │   ├── TrinketSystem.cs         // Stat modifiers
│   │   ├── BuildManager.cs          // Synergy calculation
│   │   ├── RunManager.cs            // Current run state, currencies
│   │   └── MetaProgression.cs       // Persistent upgrades (Soul Tree)
│   ├── World/
│   │   ├── IslandManager.cs         // Island loading, path selection
│   │   ├── WaveManager.cs           // Enemy wave spawning, level bounds
│   │   ├── BranchingPathUI.cs       // Route choice UI
│   │   ├── RewardSelector.cs        // Post-area ritual/gold/crystal choice
│   │   └── QuestSystem.cs           // Quest tracking, world state
│   ├── Enemy/
│   │   ├── EnemyAI.cs               // Base AI behavior
│   │   ├── EnemyAttackPatterns.cs   // Attack sequences, visual cues
│   │   ├── BossAI.cs                // Phase transitions, punish windows
│   │   └── EnemyData.cs             // SO: enemy stats, attack configs
│   ├── UI/
│   │   ├── ComboCounter.cs
│   │   ├── HealthBar.cs
│   │   ├── ManaBar.cs
│   │   ├── RitualPickerUI.cs        // Pick-1-of-N ritual selection
│   │   ├── MapUI.cs                 // Branching path visualization
│   │   └── HubUI.cs
│   └── Systems/
│       ├── SaveSystem.cs            // Meta-progression persistence
│       ├── CameraController.cs      // Side-scroll camera, zoom on stun
│       └── CoopManager.cs           // Local/online co-op
├── ScriptableObjects/
│   ├── Attacks/
│   ├── Characters/
│   ├── Rituals/
│   ├── Enemies/
│   └── Trinkets/
├── Animations/
│   ├── Characters/
│   └── Enemies/
├── Prefabs/
│   ├── Characters/
│   ├── Enemies/
│   ├── Effects/
│   └── Pickups/
└── Scenes/
    ├── Hub/
    ├── Islands/
    └── Bosses/
```

### 7.2 Combo System — State Machine Approach

```csharp
// ComboSystem.cs — Core concept
public class ComboSystem : MonoBehaviour
{
    [System.Serializable]
    public class ComboNode
    {
        public AttackData attackData;
        public Dictionary<InputType, ComboNode> branches; // Strike→next, Skill→branch, etc.
        public float inputWindow;        // Time to press next input
        public bool canDashCancel;       // Most attacks cancellable on-hit
        public bool canJumpCancel;
        public bool causesWallBounce;
        public bool causesLaunch;        // Sends enemy airborne
    }

    private ComboNode currentNode;
    private float inputTimer;
    private int hitCount;               // For Repetitive penalty tracking

    // Input buffer — store inputs for a few frames to allow pre-buffering
    private Queue<InputType> inputBuffer = new Queue<InputType>();
    private float bufferWindow = 0.1f;  // ~6 frames at 60fps

    void ProcessInput(InputType input)
    {
        if (currentNode.branches.ContainsKey(input))
        {
            var nextNode = currentNode.branches[input];
            ExecuteAttack(nextNode.attackData);
            currentNode = nextNode;
            inputTimer = nextNode.inputWindow;
        }
    }

    // On-hit confirmation enables cancel routes
    void OnAttackHit()
    {
        if (currentNode.canDashCancel) EnableDashCancel();
        if (currentNode.canJumpCancel) EnableJumpCancel();
    }
}
```

### 7.3 Deflect/Clash System

```csharp
// DefenseSystem.cs — Core concept
public class DefenseSystem : MonoBehaviour
{
    [Header("Deflect (on Dash)")]
    public float deflectWindowStart = 0.0f;   // Generous window
    public float deflectWindowEnd = 0.15f;     // ~9 frames
    
    [Header("Clash (on Skill/Heavy)")]
    public float clashWindowStart = 0.02f;    // Tighter
    public float clashWindowEnd = 0.08f;       // ~5 frames
    
    [Header("Dodge i-frames")]
    public float dodgeInvulnStart = 0.05f;
    public float dodgeInvulnEnd = 0.3f;

    private bool isDeflecting;
    private bool isClashing;

    public DamageResponse OnIncomingDamage(AttackData incomingAttack)
    {
        // Red attacks are UNSTOPPABLE
        if (incomingAttack.isUnstoppable)
        {
            if (IsInDodgeIFrames()) return DamageResponse.Dodged;
            return DamageResponse.Hit;
        }

        if (isDeflecting) return DamageResponse.Deflected;   // → Punish window opens
        if (isClashing)   return DamageResponse.Clashed;      // → Bigger punish reward
        if (IsInDodgeIFrames()) return DamageResponse.Dodged;

        return DamageResponse.Hit;
    }
}
```

### 7.4 Pressure/Stun System

```csharp
public class PressureSystem : MonoBehaviour
{
    public float maxPressure = 100f;
    public float currentPressure;
    public float punishDamageMultiplier = 2.0f;  // Red hit spark
    public float stunDuration = 3.0f;            // Free juggle time
    
    private bool isStunned;

    public void AddPressure(float amount, bool isPunishDamage)
    {
        float multiplied = isPunishDamage ? amount * punishDamageMultiplier : amount;
        currentPressure += multiplied;
        
        if (currentPressure >= maxPressure && !isStunned)
        {
            TriggerStun();
        }
    }

    void TriggerStun()
    {
        isStunned = true;
        // Camera zoom-in effect
        // Enemy becomes freely juggleable
        // After stunDuration → invincible recovery (blink white)
        StartCoroutine(StunRecovery());
    }
    
    IEnumerator StunRecovery()
    {
        yield return new WaitForSeconds(stunDuration);
        // Blink white, brief invulnerability
        // Screen close-up on enemy
        isStunned = false;
        currentPressure = 0;
    }
}
```

### 7.5 Ritual System (Roguelite Buff Stacking)

```csharp
// RitualData.cs — ScriptableObject
[CreateAssetMenu(menuName = "Roguelite/Ritual")]
public class RitualData : ScriptableObject
{
    public string ritualName;
    public RitualFamily family;         // Fire, Lightning, Water, Thorn, etc.
    public RitualCategory category;     // Core, General, Enhancement, Twin
    public RitualFamily secondFamily;   // Only for Twin rituals
    
    public int maxLevel = 3;
    public float[] basePower;           // Per level: [level1, level2, level3]
    
    // What this ritual triggers on
    public RitualTrigger trigger;       // OnStrike, OnDash, OnDeflect, OnFinisher, OnKill, etc.
    
    // What it does
    public RitualEffect effect;         // SpawnProjectile, AddBurnDamage, ChainLightning, etc.
    public GameObject effectPrefab;
}

// RitualSystem.cs — Manages active rituals during a run
public class RitualSystem : MonoBehaviour
{
    private List<ActiveRitual> activeRituals = new List<ActiveRitual>();
    private Dictionary<RitualFamily, int> familyCounts = new Dictionary<RitualFamily, int>();
    
    public void AddRitual(RitualData data)
    {
        // Check if already have this ritual → level up
        var existing = activeRituals.Find(r => r.data == data);
        if (existing != null && existing.level < data.maxLevel)
        {
            existing.level++;
            RecalculateStacking();
            return;
        }
        
        activeRituals.Add(new ActiveRitual { data = data, level = 1 });
        familyCounts[data.family] = familyCounts.GetValueOrDefault(data.family) + 1;
        RecalculateStacking();
    }
    
    // Stacking: multiple rituals boosting same mechanic increase power
    void RecalculateStacking()
    {
        // Group by shared mechanic, multiply base values
        // Ritual Power from trinkets is multiplicative
    }
    
    // Called by combat events
    public void OnCombatEvent(RitualTrigger trigger, CombatContext context)
    {
        foreach (var ritual in activeRituals)
        {
            if (ritual.data.trigger == trigger)
            {
                ExecuteRitualEffect(ritual, context);
            }
        }
    }
}
```

### 7.6 Wave Manager & Level Bounds

```csharp
// WaveManager.cs — Controls enemy spawning and camera progression
public class WaveManager : MonoBehaviour
{
    [System.Serializable]
    public class Wave
    {
        public List<EnemySpawnData> enemies;
        public Transform levelBound;     // Camera stops here until wave cleared
        public bool isOptional;          // Side path waves
    }

    public List<Wave> waves;
    private int currentWave;
    private Camera mainCamera;

    void OnWaveCleared()
    {
        currentWave++;
        if (currentWave < waves.Count)
        {
            // Move camera bound to next wave's position
            // Offer ritual/reward selection if at area end
        }
        else
        {
            // Area complete → reward selection screen
            ShowRewardSelection(); // Pick 1 of 2/3 rituals, or gold, or crystals
        }
    }
}
```

### 7.7 Branching Path / Run Structure

```csharp
// IslandManager.cs
public class IslandManager : MonoBehaviour
{
    [System.Serializable]
    public class PathNode
    {
        public string nodeName;
        public PathNodeType type;  // Combat, Shop, Challenge, Boss, Inspiration, Story
        public List<PathNode> nextNodes;  // Branching choices
        public Sprite mapIcon;
    }

    public PathNode[] island1Paths;
    public PathNode[] island2Paths;
    public PathNode[] island3Paths;
    public PathNode[] island4Paths;  // 4 islands, play 3 per run

    // After completing each island → choose next island direction
    // Fork in the road = different bosses, enemies, rituals
}
```

### 7.8 Meta-Progression (Persistent Between Runs)

```csharp
// MetaProgression.cs — Saved to disk
[System.Serializable]
public class MetaProgressionData
{
    // Currencies
    public int crystals;
    public int imbuedFruits;
    public int primordialSeeds;
    
    // Soul Tree
    public List<string> unlockedTreeNodes;  // Health+, Damage+, Revives, Rare item chance
    
    // Per-character unlocks
    public Dictionary<string, CharacterUnlocks> characterUnlocks;
    
    // Unlocked rituals (which families/rituals are in the pool)
    public List<string> unlockedRituals;
    public List<string> unlockedTwinCombinations;
    
    // Quest/world state
    public List<string> completedQuests;
    public List<string> unlockedRoutes;
}

[System.Serializable]
public class CharacterUnlocks
{
    public List<string> unlockedArcanas;
    public List<string> unlockedInspirations;
    public int characterXP;
}
```

---

## 8. ART & ANIMATION NOTES

### 8.1 Visual Style
- **Hand-drawn 2D** graphic novel / dark fantasy aesthetic
- Autumnal color palette (fall colors, wood grains, warm tones)
- Natural beauty (rebel areas) vs metallic oppression (Azra's forces)
- Smooth, sometimes comical character animations
- Lush, detailed environments

### 8.2 Animation Requirements Per Character
- Idle, Walk, Run
- Strike chain (3-4 hit combo)
- Skill/Heavy attack (with clash window frames)
- Arcana cast (per arcana type)
- Dash (forward + vertical dodge)
- Jump, Air attacks (JumpStrike, JumpSkill)
- Deflect success, Clash success
- Hit reaction, Knockdown, Get up
- Death
- Each Inspiration adds additional animation states (Dive Kick, Spinning Blade, etc.)

### 8.3 Enemy Visual Signals
- **Normal attack telegraph:** Standard animation wind-up
- **Red flash/glow:** UNSTOPPABLE attack — must dodge
- **Stun state:** White blinking, camera zoom-in
- **Punish indicator:** Red hit spark on successful punish damage

### 8.4 For Your Game (Different Art, Same Mechanics)
Since you want different characters/animations but same mechanics, use Unity's **Animator Controller** with a shared state machine template:
- Create a base Animator Controller with all combat states
- Override animations per character using **Animator Override Controllers**
- Use **Animation Events** to trigger hitbox activation, VFX, SFX at precise frames
- Consider Spine 2D or DragonBones for skeletal animation if not doing frame-by-frame

---

## 9. IMPLEMENTATION PRIORITY (Sprint Plan)

### Sprint 1: Core Combat Sandbox
1. Character movement (8-directional, jump, dash)
2. Basic Strike combo chain (3 hits + finisher)
3. Hitbox system (attack points, damage dealing)
4. Enemy takes damage, knockback, knockdown
5. Health system (player + enemy)

### Sprint 2: Defensive Depth
1. Dodge with i-frames
2. Deflect window on dash
3. Clash window on Skill/Heavy
4. Punish damage multiplier system
5. Pressure meter → Stun
6. Visual signals (red attacks, punish sparks)

### Sprint 3: Advanced Combat
1. Wall bounce physics
2. Air juggle system (launch, air combos)
3. Dash cancelling on-hit
4. Arcana (mana special) system
5. Repetitive penalty (anti-spam)
6. OTG vs Tech Hit distinction

### Sprint 4: Roguelite Layer
1. Ritual system (ScriptableObject definitions)
2. Ritual stacking & leveling
3. Post-area reward selection UI
4. Trinket system
5. Ritual trigger → effect pipeline

### Sprint 5: Run Structure
1. Wave manager (enemy spawning, level bounds)
2. Branching path UI and navigation
3. Island progression (3 of 4 islands)
4. Shop system
5. Inspiration drops from minibosses

### Sprint 6: Meta-Progression
1. Hub area (The Hearth equivalent)
2. Soul Tree upgrades
3. Currency persistence (Crystals, Fruits, Seeds)
4. Character/Arcana unlock system
5. Save/Load system

### Sprint 7: Polish & Content
1. Boss AI with phases and punish windows
2. Multiple enemy types per island
3. Mounts and companions
4. Quest system
5. Co-op (local first, then online)

---

## 10. KEY DESIGN LESSONS FROM ABSOLUM

1. **Depth through defense, not just offense.** The deflect/clash system is what elevates it above button-mashing. Players are incentivized to push INTO enemy attacks rather than retreat.

2. **Roguelite builds should feel broken.** The best runs create a feeling of unstoppable power through ritual synergies (bramble knives + echo + chain lightning = screen-clearing madness).

3. **Characters should feel incomplete at start** (controversial but intentional). The progression from basic → fully kitted is part of the roguelite dopamine loop.

4. **Fixed maps, branching paths** is a valid alternative to procedural generation. It allows hand-crafted encounters while still offering run variety.

5. **The 1-hour run length** is a double-edged sword. Long enough to build satisfying builds, but losing deep into a run is painful. Consider checkpoints or shorter run options.

6. **Elemental synergy families** (Fire/Lightning/Water/Thorn/Wind/Time/Cosmic) give players mental models for build-crafting. Players think "I'm going Lightning this run" which focuses decision-making.

7. **Anti-spam mechanics** (Repetitive penalty, combo variety rewards) force engagement with the full combat system rather than finding one optimal loop.

8. **Visual clarity in chaos** is a known weakness — especially in co-op. Invest heavily in readable VFX that don't obscure gameplay.
