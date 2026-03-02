# Coding Standards

## Naming Conventions

- **Classes/Methods:** PascalCase (`ComboSystem`, `TakeDamage`)
- **Fields:** camelCase with `[SerializeField]` (`maxHealth`, `dashSpeed`)
- **Constants:** UPPER_SNAKE (`MAX_COMBO_HITS`, `DEFAULT_BUFFER_FRAMES`)
- **Interfaces:** I-prefix (`IDamageable`, `IBuffProvider`)
- **Enums:** PascalCase values (`DamageType.Fire`, `PathType.Warden`)
- **ScriptableObjects:** PascalCase with descriptive suffix (`BrutorBaseStats`, `WardenPath`)
- **Files:** Match class name exactly (`CharacterController2D.cs`)

## Architecture Rules

- **No singletons.** Use `[SerializeField]` injection or SO-based event channels
- **ScriptableObjects for ALL data.** Attacks, paths, rituals, enemies, trinkets
- **Animation Events for timing.** Hitbox activation, VFX, SFX — never in Update()
- **Rigidbody2D for physics.** Knockback, launch, gravity — never transform.position
- **Interface-only coupling.** Pillars talk through `Shared/Interfaces/` only
- **Plain C# for testable logic.** Calculators, state machines, math as non-MonoBehaviour

## Code Style

- Strict C# with nullable annotations where useful
- `[Header("Section")]` groups in Inspector for readability
- `[Tooltip("Explanation")]` on non-obvious serialized fields
- `[Range(min, max)]` on numeric fields with known bounds
- Comments: WHY, not WHAT. Every public API has `<summary>` XML doc

## Cross-Pillar Rules

- **NEVER import from another pillar's namespace directly**
- Combat code imports from `Shared.Interfaces`, never from `Roguelite.RitualSystem`
- If you need something from another pillar, it goes through an interface
- Interface changes require ALL 3 devs to review

## Error Handling

- Combat frame code must NEVER throw — use fallback values (1.0× multiplier, 0 damage, etc.)
- Log warnings for unexpected states, don't crash
- Null-check serialized fields in Awake() with descriptive error messages

## Testing

- Plain C# classes get NUnit tests (`RitualStackCalculator`, `CharacterStatCalculator`)
- Mock interfaces using NSubstitute
- Test math with concrete numeric examples (include in test names)
- MonoBehaviour testing limited to integration scenes
