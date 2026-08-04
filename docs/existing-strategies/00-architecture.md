# 00 — Architecture: the shape every raid strategy must have

Every instance under [src/Ai/Raid/](../../src/Ai/Raid/) is the same seven-part shape. Deviating
from it is the fastest way to get a PR bounced, and it is also how you lose an hour wondering
why your action never fires.

## The seven parts

```
src/Ai/Raid/<X>/
  <X>Strategy.h/.cpp        Strategy: wires triggers -> actions, and registers multipliers
  <X>Triggers.h/.cpp        Trigger:  "is this situation true for THIS bot right now?"
  <X>TriggerContext.h       name -> Trigger factory
  <X>Actions.h/.cpp         Action:   "do the thing"   (split into Action/<X>Actions_<Boss>.cpp when large)
  <X>ActionContext.h        name -> Action factory
  <X>Multipliers.h/.cpp     Multiplier: scale/suppress OTHER actions' relevance
  <X>Helpers.h/.cpp         spell/npc/item enums, Positions, shared state maps, query helpers
  <X>Scripts.h/.cpp         (optional) server-side ScriptMgr hooks that stamp instance state
```

Naming varies slightly by age — `Aq40BossHelper.h` / `NaxxBossHelper.h` instead of
`<X>Helpers.h`, `Util/` subdirectories in AQ40/TK/Hyjal/Mag/Uld, `Action/` subdirectories in
AQ40/ICC/Naxx/RS. Match the instance you are working in, not the newest one elsewhere.

## Registration checklist

A new action that does nothing is nearly always a missed step here.

1. **Class** in `<X>Actions.h`, implementation in `<X>Actions.cpp`.
2. **Factory** in `<X>ActionContext.h`:
   ```cpp
   creators["supremus kite boss"] = &RaidBlackTempleActionContext::supremus_kite_boss;
   // ...
   static Action* supremus_kite_boss(PlayerbotAI* ai) { return new SupremusKiteBossAction(ai); }
   ```
3. **Trigger** class + factory, the same way, in `<X>Triggers.h/.cpp` and `<X>TriggerContext.h`.
4. **Wire** them in `<X>Strategy.cpp::InitTriggers`:
   ```cpp
   triggers.push_back(new TriggerNode("supremus boss is fixated on bot", {
       NextAction("supremus kite boss", ACTION_EMERGENCY + 7) }));
   ```
5. **Multipliers** (if any) in `<X>Strategy.cpp::InitMultipliers`.
6. The strategy name itself is already registered — `RaidStrategyContext.h` maps
   `"blacktemple"` → `RaidBlackTempleStrategy`, and
   `PlayerbotAI::ApplyInstanceStrategies` ([src/Bot/PlayerbotAI.cpp](../../src/Bot/PlayerbotAI.cpp))
   maps `mapId` → strategy name. A **new instance** needs entries in both, plus the
   `allInstanceStrategies` list in the same function (that list is what *removes* stale
   strategies on zoning; omitting it leaves the strategy attached after the bot leaves).

Names are global strings. A typo between `creators["…"]` and `NextAction("…")` fails silently.

## Relevance

From [src/Bot/Engine/Strategy/Strategy.h](../../src/Bot/Engine/Strategy/Strategy.h):

| Constant | Value | Used for |
|---|---|---|
| `ACTION_IDLE` | 0 | |
| `ACTION_DEFAULT` | 5 | |
| `ACTION_NORMAL` | 10 | |
| `ACTION_HIGH` | 20 | |
| `ACTION_MOVE` | 30 | generic movement, follow, formation |
| `ACTION_INTERRUPT` | 40 | |
| `ACTION_DISPEL` | 50 | |
| `ACTION_RAID` | 60 | **the default band for encounter logic** |
| `ACTION_EMERGENCY` | 90 | avoid-the-thing-that-kills-you |

House style is `ACTION_RAID + n` / `ACTION_EMERGENCY + n` with small `n` to order within a band.
Observed usage:

- `ACTION_RAID + 1` — positioning, target choice, ordinary encounter behaviour.
- `ACTION_RAID + 2/3` — pull-time misdirects, things that must beat ordinary positioning.
- `ACTION_EMERGENCY + 1..7` — hazard avoidance, break-the-debuff moves.
- `ACTION_EMERGENCY + 10/11` — state bookkeeping (`manage … timer`, `erase timers and trackers`).
  These sit at the top because they must run before anything reads the state they maintain,
  and they return `false` so they do not consume the tick (see below).

## The return-value contract

`Action::Execute` returning `true` means "I consumed this tick". Returning `false` hands the
tick to the next-most-relevant action.

This is the single most misunderstood thing in the codebase:

- A bookkeeping action (`manage phase timer`) returns **`false`** so the bot still fights.
  See `SupremusManagePhaseTimerAction::Execute`.
- A positioning action that returns `false` once the bot is in place will have the tick taken
  by `FollowAction` / `CombatFormationMoveAction`, which walk it straight off the spot. You
  then re-qualify, walk back, and the raid mills about forever. **Suppressing generic movement
  in a multiplier is the fix, not changing the return value** — see
  [08-gotchas.md](08-gotchas.md#g1-your-formation-will-not-hold-because-follow-runs-in-a-separate-pass).

## Multipliers

A `Multiplier` returns a float that scales an action's relevance. `0.0f` disables it.

```cpp
float SomeMultiplier::GetValue(Action* action)
{
    if (/* encounter not live */) return 1.0f;      // always bail out cheaply first
    if (dynamic_cast<FollowAction*>(action)) return 0.0f;
    return 1.0f;
}
```

Two dialects coexist; both are fine, and large instances use both:

- **`dynamic_cast<SomeAction*>(action)`** — type-based. Precise, but needs the header, and
  bloats the include list (`BTMultipliers.cpp` includes eleven class-action headers).
- **`action->getName() == "aq40 twin tank"`** — name-based. AQ40 uses prefix helpers
  (`IsAq40Action`, `IsTwinAction`) so a whole family can be suppressed at once. Cheaper to
  write, silently breaks on rename.

Ulduar has **no multipliers at all** — it orders everything by relevance instead. That works,
but it means Ulduar cannot suppress the generic movement/targeting engine, which is why several
Ulduar actions are `TeleportTo` "cheats" rather than movement.

## Value lookups

```cpp
AI_VALUE(GuidVector, "attackers")                    // things attacking us
AI_VALUE(GuidVector, "possible targets no los")      // hostiles regardless of LoS
AI_VALUE(GuidVector, "nearest hostile npcs")
AI_VALUE(GuidVector, "nearest triggers")             // trigger NPCs (hazard markers etc.)
AI_VALUE(Unit*, "current target")
AI_VALUE2(Unit*, "find target", "supremus")          // BY LOWERCASE NAME
AI_VALUE2(bool, "has aggro", "current target")
```

`"find target"` matches on **lowercased creature name**, not entry. It is the dominant idiom in
this tree, and its failure modes are in [08-gotchas.md](08-gotchas.md#g6-find-target-is-name-based).
Entry-based lookup exists where it matters: `GetFirstAliveUnitByEntry` (`RaidBossHelpers.cpp`),
`Aq40BossHelper::Twin::FindUnitByEntry`.

## Shared layer

[src/Ai/Raid/RaidBossHelpers.h](../../src/Ai/Raid/RaidBossHelpers.h) — the cross-instance library.
**Check here first.** It has raid markers, tracker-bot election, tank lookup, entry search,
nearest-player search, and dynamic-object search. Anything you would write twice belongs here.

[src/Ai/Base/Actions/MovementActions.h](../../src/Ai/Base/Actions/MovementActions.h) — the movement
primitives (`MoveTo`, `MoveNear`, `MoveAway`, `FleePosition`, `JumpTo`, `Follow`, `ChaseTo`) and
reusable movement actions (`AvoidAoeAction`, `RearFlankAction`, `CombatFormationMoveAction`,
`DisperseSetAction`, `RotateAroundTheCenterPointAction`, `MoveAwayFromCreatureAction`,
`MoveAwayFromPlayerWithDebuffAction`).

[src/Ai/Base/Actions/BossAuraActions.h](../../src/Ai/Base/Actions/BossAuraActions.h) — the
resistance-aura family, parameterised by boss name. You almost never need to write a new
resistance action; you need one line in an `ActionContext`:

```cpp
static Action* bwl_firemaw_fire_resistance_action(PlayerbotAI* ai)
{ return new BossFireResistanceAction(ai, "firemaw"); }
```
