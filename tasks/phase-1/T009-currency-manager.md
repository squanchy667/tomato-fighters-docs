# T009: CurrencyManager

## Metadata
| Field | Value |
|-------|-------|
| **Phase** | 1 — Foundation |
| **Type** | implementation |
| **Priority** | P1 |
| **Owner** | Dev 2 |
| **Agent** | roguelite-agent |
| **Depends On** | T001 |
| **Blocks** | T038 |
| **Status** | PENDING |
| **Branch** | `pillar2/T009-currency-manager` |

## Objective
Create the central currency management system handling 3 distinct currency types with event-driven change notifications and thread-safe modifications. This is the economic backbone that the meta-progression, soul tree, and in-run reward systems all depend on.

## Context
Tomato Fighters has 3 currencies with different persistence and rarity behaviors:

| Currency | Rarity | Persistence | Primary Use |
|----------|--------|-------------|-------------|
| **Crystals** | Common | Persist between runs | Soul Tree upgrades, hub shop |
| **Imbued Fruits** | Rare | Per-run only (reset on run end) | In-run power-ups, ritual upgrades |
| **Primordial Seeds** | Rare | Per-run only (reset on run end) | Permanent unlocks (Arcanas, Inspirations) |

The CurrencyManager is a central authority — no other system should directly modify currency values. All modifications go through this manager, which validates the operation and fires change events. This enables the UI (HUD, shop screens) to reactively update and prevents race conditions in concurrent event processing.

The project uses a no-singletons architecture. The CurrencyManager should be injectable (referenced via SerializeField or dependency injection), not accessed via a static instance.

## Requirements
1. Create `CurrencyManager.cs` in `Roguelite/`
2. Define a `CurrencyType` enum (if not already in T001 shared enums):
   - `Crystals`
   - `ImbuedFruits`
   - `PrimordialSeeds`
3. Define a `CurrencyChangeEventData` struct:
   - `currencyType` — which currency changed
   - `previousAmount` — value before the change
   - `newAmount` — value after the change
   - `delta` — the change amount (positive for add, negative for remove)
4. Implement core methods:
   - `bool TryAdd(CurrencyType type, int amount)` — Add currency, returns true on success. Amount must be > 0
   - `bool TryRemove(CurrencyType type, int amount)` — Remove currency if sufficient balance, returns false if insufficient. Amount must be > 0
   - `int GetBalance(CurrencyType type)` — Current balance for a currency
   - `bool CanAfford(CurrencyType type, int cost)` — Check if balance >= cost without modifying
   - `void ResetPerRunCurrencies()` — Reset Imbued Fruits and Primordial Seeds to 0 (called on run start)
   - `void SetBalance(CurrencyType type, int amount)` — Direct set for save/load (fires event)
5. Fire `OnCurrencyChanged` event on every successful modification (Add, Remove, Set, Reset)
6. Use C# events or Unity Events (prefer C# `event Action<CurrencyChangeEventData>` for type safety)
7. Include a persistence flag per currency type:
   - `Crystals` — `persistsBetweenRuns = true`
   - `ImbuedFruits` — `persistsBetweenRuns = false`
   - `PrimordialSeeds` — `persistsBetweenRuns = false`
8. Thread-safe modifications using `lock` on the currency store
9. Validate all inputs: reject negative amounts, reject operations that would result in negative balances
10. No maximum cap on currency values (design may add caps later via configuration)

## File Plan
| File | Description |
|------|-------------|
| `Assets/Scripts/Roguelite/CurrencyManager.cs` | Central currency authority with Add/Remove/Check methods, events, thread safety |
| `Assets/Scripts/Roguelite/CurrencyChangeEventData.cs` | (Optional, can be nested) Event data struct for currency change notifications |

## Implementation Notes
- **No singletons** — CurrencyManager is a MonoBehaviour (needs to live on a GameObject for Inspector configuration and event wiring) but is NOT accessed via `Instance`. Other systems receive it via `[SerializeField]` injection or a service locator pattern if the project adopts one
- **Thread safety via `lock`** — While Unity is primarily single-threaded, the CurrencyManager may be called from async operations (save/load, network in future co-op). Use `private readonly object _lockObj = new object()` and wrap all balance modifications in `lock(_lockObj)`
- **Events fire after modification, not before** — Listeners receive the already-committed state. This prevents listeners from interfering with the transaction
- **ResetPerRunCurrencies fires individual events** — Reset fires one `OnCurrencyChanged` event per currency being reset (ImbuedFruits and PrimordialSeeds), so UI can update each bar independently
- **Persistence flag is metadata, not logic** — The `persistsBetweenRuns` flag is read by `SaveSystem` (T039) to determine what to serialize. CurrencyManager itself doesn't handle save/load — it provides `GetBalance`/`SetBalance` for the SaveSystem to use
- **Currency amounts are always integers** — No fractional currencies. This simplifies display and prevents floating-point drift
- **Consider a `Dictionary<CurrencyType, int>` backing store** — Cleaner than 3 separate int fields, and easily extensible if new currencies are added

## Acceptance Criteria
- [ ] 3 currency types managed (Crystals, Imbued Fruits, Primordial Seeds)
- [ ] `OnCurrencyChanged` event fired on every successful currency modification
- [ ] `TryAdd` / `TryRemove` / `GetBalance` / `CanAfford` methods implemented
- [ ] `TryRemove` returns false for insufficient balance (no negative balances)
- [ ] `ResetPerRunCurrencies` zeroes per-run currencies and fires events
- [ ] Persistence flag per currency type (`Crystals` = true, others = false)
- [ ] Thread-safe modifications via `lock`
- [ ] Input validation: rejects negative amounts, zero amounts
- [ ] No singleton pattern — injectable via SerializeField
- [ ] Compiles with zero warnings

## References
- [CHARACTER-ARCHETYPES.md](/design-specs/CHARACTER-ARCHETYPES.md) — Stat Growth section mentioning currency types
- [TASK_BOARD.md](/TASK_BOARD.md) — T009 entry
- T001 (dependency) — Shared enums (CurrencyType if defined there)
- T038 (blocked) — MetaProgression + SoulTree consumes CurrencyManager for spending Crystals
- T039 (related) — SaveSystem reads/writes currency balances through CurrencyManager
- T040 (related) — HubManager displays currency in the between-run hub
