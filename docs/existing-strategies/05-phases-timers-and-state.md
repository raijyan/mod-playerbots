# 05 — Phases, timers and shared state

---

## S1. Determining the phase

In descending order of preference. **Go as far down this list as you must, and no further.**

### 1. Unit flags — most robust

```cpp
bool IsPhaseOne() { return _unit && _unit->HasUnitFlag(UNIT_FLAG_NON_ATTACKABLE); }
bool IsPhaseTwo() { return _unit && !_unit->HasUnitFlag(UNIT_FLAG_NON_ATTACKABLE); }
```
(`KelthuzadBossHelper`, `NaxxBossHelper.h`)

`UNIT_FLAG_NOT_SELECTABLE` is the other workhorse — `Aq40BossHelper::FindBurrowedOuro` uses it
to detect a submerged boss.

### 2. Auras on the boss

```cpp
Aq40SpellIds::HasAnyAura(botAI, huhuran, { Aq40SpellIds::HuhuranFrenzy })
```

### 3. Health thresholds

```cpp
bool const poisonPhase = huhuran->GetHealthPct() <= 32.0f || HasAnyAura(botAI, huhuran, {HuhuranFrenzy});
```
(`Aq40HuhuranMultiplier`) — always OR'd with an aura test, because a health threshold alone
flickers on the boundary and is wrong after a heal.

### 4. Movement/derived state

```cpp
bool IsPhaseGround() { return _unit && !_unit->IsFlying(); }
bool IsPhaseFlight() { return _unit &&  _unit->IsFlying(); }
```
(`SapphironBossHelper`) — which also latches the *transition*, since "just landed" is what the
positioning actually keys off:

```cpp
bool now_flying = _unit->IsFlying();
if (_was_flying && !now_flying) _last_land_ms = getMSTime();
_was_flying = now_flying;
// …
bool JustLanded() { return _last_land_ms && getMSTime() - _last_land_ms <= 5000; }
```

### 5. Composite — the full worked example

`BlackTempleHelpers::GetIllidanPhase(Unit*)` derives six phases from flags, auras, health and
the boss's MotionMaster destination, with no core coupling at all:

```cpp
if (illidan->GetHealth() == 1 || illidan->HasAura(SPELL_SHADOW_PRISON))        return -1;
// transition: heading to (or standing on) the landing spot, and not selectable
if (dest.GetExactDist2d(ILLIDAN_LANDING_POSITION) < 0.2f && HasUnitFlag(NOT_SELECTABLE)) return 0;
if (illidan->HasUnitFlag(UNIT_FLAG_NOT_SELECTABLE))                            return 2;   // flying
if (illidan->GetHealthPct() > 65.0f)                                           return 1;
if (!HasAura(CAGED) && (HasAura(DEMON_FORM) || HasAura(DEMON_TRANSFORM_1..3)))  return 4;
if (illidan->GetHealthPct() > 30.0f)                                           return 3;
return 5;
```

**This is the pattern to reach for.** It is more code than a `dynamic_cast` to the boss AI, and
it is worth it, because it does not break when the core changes.

### 6. Reading the boss's own AI — last resort, fragile

Two forms exist in the tree. Both work. Both are liabilities; see
[10-antipatterns-and-pr-risks.md](10-antipatterns-and-pr-risks.md).

**EventMap access** — `GenericBossHelper<BossAiType>` in `NaxxBossHelper.h`:

```cpp
_ai = dynamic_cast<BossAiType*>(_target->GetAI());
_event_map = &_ai->events;                       // reads the boss's scheduled events
```

**Copied class declaration** — `TK/Util/TKKaelthasBossAI.h` redeclares the core's
`struct boss_kaelthas` (which lives inside a .cpp and is not exported) with an added accessor:

```cpp
uint32 GetPhase() const { return _phase; } // This is the only addition to the existing class
```

then `TKMultipliers.cpp` does `dynamic_cast<boss_kaelthas*>(kaelthas->GetAI())->GetPhase()`.
This is an ODR violation that works only because the member layout happens to match the core's.
**Do not copy it into a new instance.** Derive the phase (option 5) instead.

---

## S2. Where state lives

Three storage locations, three lifetimes. Pick deliberately.

### Per-instance, in `<X>Helpers.cpp`

```cpp
extern std::unordered_map<uint32 /*instanceId*/, time_t> supremusPhaseTimer;
extern std::unordered_map<uint32, ObjectGuid>            eastFlameGuid;
extern std::unordered_map<uint32, int>                   illidanLastPhase;
```

Plain globals, written from bot AI code, which runs on the map update thread. Not mutex-guarded.
Acceptable *only* because all bots in one instance update on the same thread — do not read
these from a `ScriptMgr` hook on another map's thread.

### Per-bot, in `<X>Helpers.cpp`

```cpp
extern std::unordered_map<ObjectGuid, TankPositionState> shahrazTankStep;
extern std::unordered_map<ObjectGuid, size_t>            flameTankWaypointIndex;
extern std::unordered_map<ObjectGuid, Position>          illidanShadowTrapDestination;
extern std::unordered_set<ObjectGuid>                    hasReachedAkamaChannelerPosition;
```

### Per-instance, in `<X>Scripts.cpp` — mutex-guarded

When the state is written by a `ScriptMgr` hook (see
[06-server-side-hooks-and-cheats.md](06-server-side-hooks-and-cheats.md)) it *is* guarded,
because hooks fire from map threads:

```cpp
namespace { std::unordered_map<uint32, IccInstanceState> g_state; std::mutex g_stateMutex; }

IccInstanceState& IccState(uint32 instanceId)
{
    std::lock_guard lock(g_stateMutex);
    return g_state[instanceId];
}
```
(`ICCScripts.cpp`; `Aq40Scripts.cpp` and `RSScripts.cpp` are the same shape.)

> Note `IccState` returns a reference **after** releasing the lock, so callers mutating the
> returned struct are unsynchronised. `Aq40Scripts.cpp` avoids this by taking the lock in each
> accessor and returning by value. Prefer the AQ40 shape.

### Caching hot lookups

`Aq40BossHelper::Detail` caches the group walk in a `thread_local` snapshot keyed by
`(botGuid, mapId, instanceId, getMSTime())`. Copy this when a helper is called from a
multiplier — multipliers run once per candidate action, so a naive group walk runs dozens of
times per bot per tick.

---

## S3. Timers

**Stamp on first observation**, via a low-cost action wired at `ACTION_EMERGENCY + 10` that
returns `false`:

```cpp
bool SupremusManagePhaseTimerAction::Execute(Event)
{
    Unit* supremus = AI_VALUE2(Unit*, "find target", "supremus");
    if (!supremus) return false;
    supremusPhaseTimer.try_emplace(supremus->GetMap()->GetInstanceId(), std::time(nullptr));
    return false;                       // never consume the tick
}
```

**Two clocks are in use and they are not interchangeable:**

- `std::time(nullptr)` → `time_t`, whole seconds. Fine for 5s DPS waits.
- `getMSTime()` / `getMSTimeDiff(then, now)` → `uint32` ms, **wraps**. Always use
  `getMSTimeDiff`, never `now - then`, and treat `0` as "never stamped":

```cpp
bool IsRecent(uint32 stampedAtMs, uint32 windowMs, uint32 nowMs)
{
    return stampedAtMs && getMSTimeDiff(stampedAtMs, nowMs) <= windowMs;
}
```
(`Aq40Scripts.cpp`)

**Deduplicating repeated events.** The Twins cast their teleport spell twice per swap; a dedupe
window collapses them into one sequence number:

```cpp
if (!IsRecent(state.lastTeleportAtMs, kTwinTeleportDedupeMs /*5000*/, nowMs))
    ++state.teleportSequence;
state.lastTeleportAtMs = nowMs;
```

**Derived phase clock from a single stamp** — no per-phase events needed:

```cpp
const int groupIndex = (elapsed % 30) / 10;                 // 3 groups, 10s each
```
(`GetGurtoggActiveRotationGroup`)

**Deferred single action:** `botAI->AddTimedEvent(lambda, delayMs)`. Capture **GUIDs, not
pointers**, and re-resolve inside:

```cpp
const ObjectGuid spineGuid = spineGo->GetGUID();
botAI->AddTimedEvent([this, spineGuid]()
{
    if (GameObject* targetSpine = botAI->GetGameObject(spineGuid))
        targetSpine->Use(bot);
}, urand(1000, 2000));
```
(`HighWarlordNajentusRemoveImpalingSpineAction::Execute`. The `urand` jitter also staggers the
raid so 25 bots do not all click on the same tick.)

---

## S4. Cleaning up

Every map you add must be erased, or a wipe leaves the raid acting on last attempt's state.

The convention is one `<instance> erase timers and trackers` action fired by a
`<instance> bot is not in combat` trigger at `ACTION_EMERGENCY + 11`:

```cpp
bool BlackTempleBotIsNotInCombatTrigger::IsActive()
{
    return bot->GetMapId() == BLACK_TEMPLE_MAP_ID && !AI_VALUE2(bool, "combat", "self target");
}
```

```cpp
if (!AI_VALUE2(Unit*, "find target", "supremus") && supremusPhaseTimer.erase(instanceId) > 0)
    erased = true;
```
(`BlackTempleEraseTimersAndTrackersAction::Execute`)

Points to copy:

- **Per-boss gating.** Only erase a boss's state when *that boss* is not present, so leaving
  Supremus does not wipe Illidan bookkeeping.
- **Role-partitioned.** BT's version branches on tank / healer / dps so each bot only touches
  the maps it owns, keeping the not-in-combat path cheap.
- **`erase(...) > 0` to compute the return value.** The action returns `true` only if it
  actually did something, so an idle raid does not consume a tick every frame.
- Existing actions: `black temple erase timers and trackers`,
  `aq40 erase timers and trackers`, `serpent shrine cavern erase timers and trackers`,
  `hyjal summit erase trackers`, `magtheridon erase timers and trackers`, `void reaver erase trackers`.

**Also clean the markers.** State is not just maps — see
[02-coordination-and-assignment.md](02-coordination-and-assignment.md#c2-raid-target-icons).
`Aq40Helpers::ResetEncounterState` erases C'Thun, Skeram and Twin state together and logs it.

**And free on map destruction**, for anything held in a `Scripts.cpp` instance map:

```cpp
class IccMapCleanupScript : public AllMapScript
{
    void OnDestroyMap(Map* map) override { IcecrownHelpers::IccResetInstance(map->GetInstanceId()); }
};
```
Without this, `g_state` grows by one entry per instance created, forever.

---

## S5. Trigger check intervals

`Trigger(botAI, name, checkInterval = 1)`. The default re-evaluates every tick. If your trigger
does a group walk plus a grid search, pass a larger interval — nothing in `src/Ai/Raid/`
currently does, which is a latent cost, not a correctness problem.
