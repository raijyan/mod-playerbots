# 10 — Antipatterns and PR risks

Two lists. The first is what a reviewer will object to in *your* diff. The second is what is
already in the tree and should **not** be treated as precedent.

---

## Part 1 — what will get picked apart

### A1. A novel mechanism where a canonical one exists

The most common and most avoidable objection. Before writing:

- a group walk to find the tank → `GetGroupMainTank` / `Aq40BossHelper::GetEncounterPrimaryTank`
- a "who does this job" election → `IsMechanicTrackerBot`
- a raid icon setter → `MarkTargetWithSkull`
- a resistance action → `BossFireResistanceAction(ai, "<boss>")`
- a nearest-player search → `GetNearestPlayerInRadius`
- a dynamic-object AoE search → `GetDynamicObjectPositions`
- a rear-arc mover → `RearFlankAction`
- a safe-position search → `KarazhanHelpers::TryFindSafePositionWithSafePath`
- a spread-out action → `avoid aoe` / `disperse ranged` / `DisperseSetAction`
- a cooldown-suppression list → `IsDpsCooldownAction` in `BTMultipliers.cpp`

If none fits, say so in the PR body and say why. "I didn't know it existed" is the outcome this
guidebook exists to prevent.

### A2. Duplicating a helper into a new instance instead of promoting it

If two instances need it, it belongs in
[RaidBossHelpers.h](../../src/Ai/Raid/RaidBossHelpers.h) or
[src/Ai/Base/Actions/](../../src/Ai/Base/Actions/), not copy-pasted. The tree already has
avoidable near-duplicates: `SupremusMoveAwayFromVolcanosAction::FindSafestNearbyPosition`,
`KarazhanHelpers::TryFindSafePositionWithSafePath` and
`ZulAmanHelpers::FindSafestNearbyPosition` are three implementations of one algorithm. Do not
add a fourth.

Same for `GetIllidanWarlockTank` / `GetZerevorMageTank` — identical
"assistant of class X, else any bot of class X" logic, written twice.

### A3. Hard-coded roster sizes

`(healers.size() + 1) / 2`, not `healers[0..3]`. See
[08-gotchas.md](08-gotchas.md#g11-do-not-cap-a-roster-at-a-fixed-size).

### A4. Unbounded or unfreed state

Every `std::unordered_map` you add needs:
- an erase path in the instance's `erase timers and trackers` action, **and**
- an `AllMapScript::OnDestroyMap` reset if it lives in a `Scripts.cpp`, **and**
- a bounded size if it accumulates per-event entries (`ICCScripts.cpp` evicts by age).

### A5. Unsynchronised state touched by a `ScriptMgr` hook

Hooks run on map threads. Take the mutex for the whole accessor and return by value —
`Aq40Scripts.cpp`'s shape, not `ICCScripts.cpp`'s reference-returning `IccState`. See
[06-server-side-hooks-and-cheats.md](06-server-side-hooks-and-cheats.md#k7-thread-safety).

### A6. Long-lived raw `Player*` / `Unit*`

Project rule (`AGENTS.md`): do not store a raw pointer past the current call. Store the
`ObjectGuid` and re-resolve. In practice this matters most in `AddTimedEvent` lambdas — capture
GUIDs, resolve inside:

```cpp
const ObjectGuid najentusGuid = najentus->GetGUID();
botAI->AddTimedEvent([this, najentusGuid]() { Unit* u = botAI->GetUnit(najentusGuid); /* … */ }, delay);
```

The `<X>Helpers` state maps correctly store `ObjectGuid`, not pointers
(`eastFlameGuid`, `illidanShadowTrapGuid`). Follow that.

### A7. Style

The module ships `./code_format.sh` (clang-format over every `.cpp`/`.h`); AzerothCore's
`apps/codestyle/codestyle-cpp.py` and CI `-Werror` enforce the rest. The ones that actually get
flagged in this tree:

- `auto const&`, not `const auto&`. `Type const*`, not `const Type*`.
- Allman braces; no braces around single-line statements; `if (x)` not `if(x)`.
- 4-space indent, 120 columns, LF, trailing newline.
- `{}` format specifiers in `LOG_*`, never `%u`/`%s`.
- `urand`/`frand`/`rand_chance` from `Random.h`, never `std::rand`.
- Typed helpers: `IsPlayer()` not `GetTypeId() == TYPEID_PLAYER`.

Note the existing raid code is **inconsistent** on `const auto&` vs `auto const&` and on
`const Position&` vs `Position const&`. Match the project rule in new code even where the
surrounding file does not.

### A8. Reaching into core internals

See Part 2 below. If you need boss state, derive it — `GetIllidanPhase` is the proof that six
phases can be derived from public state with no coupling at all.

### A9. Changing the encounter to make bots win

`RSScripts.cpp` roots the boss, teleports it home, and grants bots damage immunity. If you copy
that pattern, the PR must say why the bot-side solution is not viable. It is not a neutral
choice: it changes the fight for the humans in the raid too.

### A10. Disabling something without recording why

Naxx has three encounters commented out with no note. Nobody now knows whether Heigan was
broken, slow, or superseded. If you disable, leave a one-line comment saying why and when.

---

## Part 2 — existing code that is not precedent

These are in the tree, they work, and they are liabilities. Know about them so you do not
propagate them, and so you are not surprised when they break.

### X1. `TK/Util/TKKaelthasBossAI.h` re-declares a core class — ODR violation

The core's `struct boss_kaelthas : public BossAI` is defined **inside**
`src/server/scripts/Outland/TempestKeep/Eye/boss_kaelthas.cpp`. It is not exported and has no
public phase accessor. The module declares its **own** `struct boss_kaelthas : public BossAI`
with the same name at the same scope, mirroring the private member layout:

```cpp
// module (TKKaelthasBossAI.h)          // core (boss_kaelthas.cpp)
uint32 GetPhase() const { return _phase; }
private:                                 private:
    uint32 _phase;                           uint32 _phase;
    uint8  _advisorsAlive;                   uint8  _advisorsAlive;
    bool   _transitionSceneReached = false;  bool   _transitionSceneReached = false;
```

`TKMultipliers.cpp` then does `dynamic_cast<boss_kaelthas*>(kaelthas->GetAI())->GetPhase()` at
nine call sites.

This is one program with two definitions of the same class — formally UB. It functions only
because the mangled name matches (so `dynamic_cast` succeeds) and the member layout happens to
be identical. **It breaks silently** — reading garbage, not crashing — if anyone upstream adds,
removes or reorders a member of the core struct.

The comment `// This is the only addition to the existing class` is honest about what it is.
Do not replicate it for a new boss; derive the phase from observable state instead
([05-phases-timers-and-state.md](05-phases-timers-and-state.md#s1-determining-the-phase)).

### X2. `GenericBossHelper<BossAiType>` reads the boss's `EventMap`

```cpp
_ai = dynamic_cast<BossAiType*>(_target->GetAI());
_event_map = &_ai->events;
```
(`Naxx/NaxxBossHelper.h`)

Less severe than X1 — `BossAI::events` is a real public member — but it couples bot behaviour
to the core script's internal event scheduling, which upstream changes freely and without
notice. The equivalent read in `UldBossHelper.h` is **commented out**, which suggests it was
tried and abandoned there.

Note that none of the Naxx helpers that actually ship (`Kelthuzad`, `Razuvious`, `Sapphiron`,
`Gluth`, `Loatheb`, `FourHorsemen`, `Thaddius`) derive from `GenericBossHelper` — they all
derive phase from flags, auras, health and casts. The template is effectively dead weight.
**Follow the ones that ship.**

### X3. `ICCScripts.cpp::IccState` hands out an unlocked reference

```cpp
IccInstanceState& IccState(uint32 instanceId)
{
    std::lock_guard lock(g_stateMutex);
    return g_state[instanceId];          // lock released here; caller mutates unlocked
}
```

Callers do `IccState(id).malleableGoo.push_back(...)` from spell hooks on map threads. The
`g_state` map itself is protected against concurrent rehash, but the per-instance struct is not.
`Aq40Scripts.cpp` gets this right; copy that.

### X4. One cheat trigger strips follow before checking whether cheats are on

Ulduar's cheats are properly gated — `HasCheat(BotCheatMask::raid)` is the first line of
`MimironCheatTrigger`, `VezaxCheatTrigger` and `YoggSaronOminousCloudCheatTrigger`. Gating in
the trigger is the right place, and worth copying.

The exception is `YoggSaronBossRoomMovementCheatTrigger::IsActive`, which returns `true`
**before** its `HasCheat` check whenever the bot still has `FollowMasterStrategy`, so that the
action can strip follow. The action then bails at `!HasCheat`. With cheats off that leaves
follow removed with no restore on this path. Do not copy the ordering; check the gate first.

### X5. Ulduar has no multipliers

`Uld/` has no `UldMultipliers.*`. Everything is ordered by relevance alone, which means Ulduar
cannot suppress generic movement or generic targeting. That is a plausible reason several
Ulduar actions are `TeleportTo` rather than pathing — a teleport cannot be argued with by
`FollowAction`. If you are adding Ulduar behaviour that needs to hold a position, you will
probably need to add the multiplier file rather than reaching for another teleport.

### X6. `AI_VALUE2(Unit*, "find target", "name")` everywhere

Name-based lookup is the dominant idiom, and it is fragile — see
[08-gotchas.md](08-gotchas.md#g6-find-target-is-name-based). It is too entrenched to change
now, but for a *new* encounter prefer entry-based lookup where correctness matters
(`GetFirstAliveUnitByEntry`), and keep names for readability in triggers where a miss is benign.

### X7. Stubs and dead code

- `Aq40Helpers::ShouldSuppressTwinPrePullMaintenance(...)` — unconditionally `return false;`
  with `(void)` casts on all parameters.
- `NaxxActions_Heigan.cpp` — entirely commented out.
- `HeiganDanceMultiplier`, `GothikGenericMultiplier`, three Patchwerk triggers — commented out
  in `NaxxStrategy.cpp`.
- `MovementActions.h` has a commented-out `SearchBestGroundZForPath` declaration.

---

## Pre-PR checklist

- [ ] Every helper I wrote is not already in `RaidBossHelpers.h` or `src/Ai/Base/Actions/`.
- [ ] Every action name in `NextAction(...)` matches a `creators["..."]` key exactly.
- [ ] New instance? → `RaidStrategyContext.h`, the `mapId` switch **and** the
      `allInstanceStrategies` list in `PlayerbotAI::ApplyInstanceStrategies`.
- [ ] New `Scripts.cpp`? → declared and called in `src/Script/Playerbots.cpp`.
- [ ] Bookkeeping actions return `false`.
- [ ] Every new map has an erase path, and an `OnDestroyMap` reset if hook-owned.
- [ ] Every raid icon I set has a clear path.
- [ ] If I removed `FollowMasterStrategy`, there is a restore path that runs even if my action
      does not.
- [ ] Spell id tests cover 10N/25N/10H/25H.
- [ ] `getMSTimeDiff`, not subtraction.
- [ ] `GetCurrentSpell` checks the channelled slot too.
- [ ] No raw `Player*`/`Unit*` captured in a lambda or stored across ticks.
- [ ] Roster sizes derived, not hard-coded.
- [ ] Style clean: `./code_format.sh` in the module (clang-format), and
      `python apps/codestyle/codestyle-cpp.py` from the AzerothCore root.
- [ ] The PR body names which existing mechanic I reused, or why none fitted.
