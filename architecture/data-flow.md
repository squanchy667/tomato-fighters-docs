# Data Flow

## Single Combat Frame

```
Input (Strike button)
    │
    ▼
1. InputBufferSystem
    │── Queue input with timestamp
    │── Check if within buffer window (0.1s)
    │
    ▼
2. ComboSystem
    │── Current node has branch for this input?
    │── Yes → advance to next ComboNode
    │── Load AttackData from node
    │
    ▼
3. CharacterController2D
    │── Apply attack animation
    │── Lock movement during attack frames
    │
    ▼
4. HitboxManager (via Animation Event)
    │── Activate hitbox collider at configured frame
    │── OnTriggerEnter2D → detect IDamageable targets
    │── Report hit-confirm to ComboSystem
    │
    ▼
5. Damage Calculation
    │── Base ATK from CharacterStatCalculator
    │── × AttackData.damageMultiplier
    │── × IBuffProvider.GetDamageMultiplier(type)
    │── × IBuffProvider.GetPathDamageMultiplier()
    │── × Passive modifier (Bloodlust stacks, Distance Bonus, etc.)
    │── × Punish multiplier (if applicable)
    │── = Final damage
    │
    ▼
6. IDamageable.TakeDamage(DamagePacket)
    │── Enemy takes damage
    │── Pressure meter updated
    │── Knockback/Launch applied via Rigidbody2D
    │
    ▼
7. ICombatEvents.OnStrike fired
    │── RitualSystem checks all active rituals
    │── Matching triggers fire effects (burn, chain lightning, etc.)
    │── Additional on-hit effects applied
    │
    ▼
8. ComboSystem post-hit
    │── Hit confirmed → enable dash/jump cancel
    │── Reset input timer for next node
    │── Bloodlust stack +1 (if Slasher)
```

## Path Selection Flow

```
Player reaches Upgrade Shrine (end of Area 1 or 2)
    │
    ▼
1. IRunProgressionEvents fired (OnAreaCleared)
    │
    ▼
2. PathSystem checks state
    │── No Main path? → Show all 3 paths for character
    │── Has Main, no Secondary? → Show remaining 2 paths
    │── Both selected? → Skip (no shrine)
    │
    ▼
3. PathSelectionUI displays options
    │── Path name, description, T1 preview
    │── Stat bonus preview (before/after)
    │── Ability preview
    │
    ▼
4. Player confirms selection
    │
    ▼
5. PathSystem applies
    │── Set Main or Secondary path
    │── Unlock Tier 1
    │── Apply stat bonuses → CharacterStatCalculator recalculates
    │── Unlock T1 ability → IPathProvider.IsPathAbilityUnlocked returns true
    │
    ▼
6. PathAbilityExecutor (Combat Pillar) detects new ability
    │── Queries IPathProvider for active abilities
    │── Binds ability to input (if active) or applies modifier (if passive)
```

## Run Loop Flow

```
Hub (between runs)
    │── Character select → load CharacterBaseStats
    │── Arcana select → equip 1 arcana
    │── Soul Tree → apply permanent bonuses
    │
    ▼
Run Start → IRunProgressionEvents.OnRunStarted
    │
    ▼
Island 1 ─────────────────────────────────────────
    │
    ├── Area 1 (combat waves)
    │   └── WaveManager → spawn → fight → clear
    │       └── OnAreaCleared → RewardSelectorUI (pick ritual)
    │
    ├── Upgrade Shrine → PathSelectionUI (choose MAIN path)
    │
    ├── Area 2 (combat waves)
    │   └── Clear → pick ritual
    │
    ├── Upgrade Shrine → PathSelectionUI (choose SECONDARY path)
    │
    ├── Area 3 (optional: shop, challenge, story)
    │
    └── Boss → BossAI phases → defeat
        ├── OnBossDefeated → Main T2, Secondary T2 unlocked
        ├── Inspiration drop (from mini-boss earlier)
        └── OnIslandCompleted → choose next island
    │
    ▼
Island 2 ─────────────────────────────────────────
    │── 2nd Arcana unlocked
    │── More areas, rituals stack higher
    │── Boss (T3 deferred — max tier is T2)
    │
    ▼
Island 3 ─────────────────────────────────────────
    │── Final island
    │── Fully kitted character
    │── Final boss
    │
    ▼
Run End → IRunProgressionEvents.OnRunEnded
    │── Crystals persist
    │── Fruits/Seeds earned
    │── Run rituals/trinkets lost
    │── Path selections lost
    │── Return to Hub
```

## Error Handling

| Situation | Fallback |
|-----------|----------|
| No matching ritual trigger | Skip silently — don't block combat frame |
| IBuffProvider returns null | Default to 1.0× multiplier |
| PathAbility not unlocked | Button does nothing, no error |
| Enemy already dead when hit | Ignore damage, don't crash |
| Save file corrupted | Reset to defaults, warn player |
| No enemies to target (AoE) | Ability activates but hits nothing |
