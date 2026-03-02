# Test Plan

## Strategy

- **Unit tests** for pure C# classes (calculators, math, state logic)
- **Integration scenes** for MonoBehaviour interaction testing
- **Playtest sessions** for game feel and balance (weekly)
- **Mock interfaces** using NSubstitute for isolated testing

## Coverage Goals

| Module | Target | What to Test |
|--------|--------|-------------|
| CharacterStatCalculator | 90% | All stat formulas, edge cases, path + ritual combo |
| RitualStackCalculator | 90% | Level scaling, stacking, multiplicative power |
| ComboSystem | 70% | State transitions, branching, cancel flags |
| PathSystem | 80% | Selection logic, tier progression, T3 Main-only gate |
| CurrencyManager | 80% | Add/remove, persistence flags, event firing |
| EnemyAI state machine | 70% | State transitions, aggression config |

## Test Types

| Type | Tool | Location |
|------|------|---------|
| Unit | NUnit + NSubstitute | `Tests/Editor/` |
| Integration | Unity Test Runner | `Tests/PlayMode/` |
| Balance | Manual playtest | Weekly sessions |
| Performance | Unity Profiler | Phase 5-6 |

## Running Tests

```bash
# Unity Editor: Window → General → Test Runner
# Or via command line:
Unity -runTests -testPlatform EditMode -projectPath ./tomato-fighters
```

## What NOT to Test

- MonoBehaviour lifecycle (Awake, Start, Update) — test extracted logic instead
- Unity physics specifics — trust the engine
- ScriptableObject serialization — trust Unity's asset pipeline
- Animation Event timing — visual inspection in editor
