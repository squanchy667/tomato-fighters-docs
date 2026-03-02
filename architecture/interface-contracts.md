# Interface Contracts

> **THE MOST CRITICAL FILE IN THE PROJECT.**
> All cross-domain communication MUST go through these interfaces.
> Changes require ALL 3 devs to review and approve.

---

## Combat → Roguelite (Dev 1 fires, Dev 2 subscribes)

```csharp
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
    event Action<PathAbilityEventData> OnPathAbilityUsed;
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
    float GetPathDamageMultiplier();
    float GetPathDefenseMultiplier();
    float GetPathSpeedMultiplier();
    List<PathAbility> GetActivePathAbilities();
}
```

## Path Provider (Dev 2 provides, Dev 1 + Dev 3 query)

```csharp
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
    event Action<AreaClearedData> OnAreaCleared;
    event Action<BossDefeatedData> OnBossDefeated;
    event Action<IslandCompletedData> OnIslandCompleted;
    event Action<ShopEnteredData> OnShopEntered;
    event Action OnRunStarted;
    event Action<RunEndData> OnRunEnded;
    event Action<PathSelectedData> OnMainPathSelected;
    event Action<PathSelectedData> OnSecondaryPathSelected;
    event Action<PathTierUpData> OnPathTierUp;
}
```

## Shared Enums

```csharp
public enum CharacterType { Brutor, Slasher, Mystica, Viper }

public enum PathType
{
    Warden, Bulwark, Guardian,           // Brutor
    Executioner, Reaper, Shadow,         // Slasher
    Sage, Enchanter, Conjurer,           // Mystica
    Marksman, Trapper, Arcanist          // Viper
}

public enum StatType
{
    Health, Defense, Attack, RangedAttack, Speed,
    Mana, ManaRegen, CritChance,
    StunRate,       // design docs call this "PRS / Pressure Rate" — code always uses StunRate (DD-1, T006)
    CancelWindow
}

public enum DamageType { Physical, Fire, Lightning, Water, Thorn, Gale, Time, Cosmic, Necro }
public enum RitualTrigger { OnStrike, OnSkill, OnDash, OnDeflect, OnClash, OnFinisher, OnKill, OnArcana, OnJump, OnDodge, OnTakeDamage }
public enum TelegraphType { Normal, Unstoppable }
public enum RitualFamily { Fire, Lightning, Water, Thorn, Gale, Time, Cosmic, Necro }
public enum RitualCategory { Core, General, Enhancement, Twin }
```

## Shared Data Structures

```csharp
public struct DamagePacket
{
    public DamageType type;
    public float amount;
    public bool isPunishDamage;
    public Vector2 knockbackForce;
    public Vector2 launchForce;
    public CharacterType source;
}

[CreateAssetMenu(menuName = "TomatoFighters/AttackData")]
public class AttackData : ScriptableObject
{
    public float damageMultiplier;
    public Vector2 knockbackForce;
    public Vector2 launchForce;
    public AnimationClip animationClip;
    public float hitboxStartFrame;
    public float hitboxActiveFrames;
    public TelegraphType telegraphType;
    public bool causesWallBounce;
    public bool causesLaunch;
}
```

## Data Flow Summary

```
Player Input
    │
    ▼
[Combat Pillar]
    │── CharacterController2D → movement/dash
    │── InputBuffer → ComboSystem → attack
    │── HitboxManager → collision check
    │── DefenseSystem → deflect/clash/dodge
    │
    ├── FIRES ICombatEvents ──────────────────► [Roguelite Pillar]
    │                                               │── RitualSystem triggers
    │                                               │── Stacking calculation
    │                                               │── Buff values updated
    │                                               │
    ◄── QUERIES IBuffProvider ◄──────────────────── │
    │   (damage × ritual × path × trinket)
    │
    │── CALLS IDamageable.TakeDamage ────────► [World Pillar]
    │                                               │── EnemyBase takes damage
    │                                               │── EnemyAI state change
    │                                               │── BossAI phase check
    │                                               │
    │                                               ├── FIRES IRunProgressionEvents
    │                                               │       │
    │                                               │       ▼
    │                                               │   [Roguelite Pillar]
    │                                               │   │── Reward selection
    │                                               │   │── Path tier-up
    │                                               │   └── Currency update
    │
    ▼
[Frame rendered with HUD updates from World Pillar]
```
