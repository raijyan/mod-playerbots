# 01 — Movement and positioning

The single largest category, and the one that wastes the most time, because the bot has
**three independent movement systems** competing for each tick: your encounter action, the
follow strategy, and combat-formation/chase movement. Almost every "the bots won't stand
still" bug is one of the other two winning.

---

## M1. Stop bots following during a positioning phase

**The mechanism.** Zero the generic movement actions in a multiplier. Nothing else works
reliably — returning `true` from your own action does not stop follow, because follow runs in
a separate pass.

Canonical predicate, [Aq40Multipliers.cpp](../../src/Ai/Raid/Aq40/Aq40Multipliers.cpp):

```cpp
bool IsGenericMovementAction(Action* action)
{
    return dynamic_cast<CombatFormationMoveAction*>(action) ||
           dynamic_cast<FollowAction*>(action) ||
           dynamic_cast<FleeAction*>(action) ||
           dynamic_cast<MovementAction*>(action);
}
```

Note `MovementAction` is the base of the other three *and of your own action*, so an unguarded
`return 0.0f` on `IsGenericMovementAction` kills your positioning action too. Every user
excludes its own actions by name:

```cpp
if (eye && eye->IsAlive() && !inStomach && !IsAq40Action(actionName) && IsGenericMovementAction(action))
    return 0.0f;
```

**Variants in the tree, all equivalent in effect:**

| Instance | Shape |
|---|---|
| AQ40 | name-prefix exclusion — `!IsAq40Action(action->getName())` |
| ICC | explicit list — `dynamic_cast<FollowAction*>() \|\| dynamic_cast<CombatFormationMoveAction*>()` (`ICCMultipliers.cpp`) |
| BT | one multiplier per boss, e.g. `MotherShahrazControlMovementMultiplier` |
| SSC | `SSCMultipliers.cpp` — also gates on which *phase* of the tainted-core pass is live |

**Range-gating.** If the multiplier needs the boss to be *visible* to fire, it will not fire
during a pre-pull approach. AQ40 C'Thun solves this with a geometric gate that does not depend
on seeing the boss at all — a radius test around the chamber plus a Z band:

```cpp
if (dx*dx + dy*dy < 130.0f*130.0f && std::fabs(bot->GetPositionZ() - 108.9f) < 12.0f)
    return 0.0f;
```

See [08-gotchas.md](08-gotchas.md#g2-your-suppression-multiplier-does-not-apply-before-the-pull).

---

## M2. Take the follow strategy off a bot entirely (and give it back)

Heavier than M1: it removes `FollowMasterStrategy` from the non-combat engine, so it survives
across ticks and across your action not running.

```cpp
FollowMasterStrategy followMasterStrategy(botAI);
if (botAI->HasStrategy(followMasterStrategy.getName(), BotState::BOT_STATE_NON_COMBAT))
    botAI->ChangeStrategy(REMOVE_STRATEGY_CHAR + followMasterStrategy.getName(), BotState::BOT_STATE_NON_COMBAT);
```

Users: `YoggSaronBossRoomMovementCheatAction`, `YoggSaronUsePortalAction` (re-adds it),
`UldActions.cpp` (several), `Aq40Actions_Shared.cpp`.

**You must own the un-stage.** If the bot dies, zones, or the encounter resets while follow is
removed, it stays removed for the rest of the session. Two patterns cover this:

- **AQ40**: an explicit geometric un-stage — leave the zone and follow is restored
  (`Aq40Actions_Shared.cpp`, "Give follow back if C'Thun staging is not asking for it").
- **AQ40 full reset**: `Aq40Helpers::TryRecoverAq40FollowState`
  ([Aq40Helpers_Shared.cpp](../../src/Ai/Raid/Aq40/Util/Aq40Helpers_Shared.cpp)) — the most complete
  recovery in the tree. It clears the `stay` strategy in both engines, resets the `stay` /
  `return` / `random` position values, clears `last movement` / `last area trigger` /
  `last taxi`, clears `UNIT_STATE_CHASE` and `UNIT_STATE_FOLLOW`, clears the MotionMaster, and
  optionally re-executes `FollowAction`. If you have a stuck bot, this is the checklist of
  everything that can be holding it.

Prefer M1 (multiplier) over M2 (strategy removal) unless you genuinely need it to persist
outside your action's lifetime.

---

## M3. Hold a position without owning the tick

The problem: your action returns `false` once the bot has arrived (deliberately, so it can
cast), and the tick goes to follow/formation, which walks it off.

**Do not** fix this by returning `true` forever — that stops the bot fighting. Fix it with M1,
and let the action stay live and simply hold. The comment on `Aq40BossHelper::Twin::ShouldStageForPull`
spells it out:

> No inner cut-off. Dropping out of this action on arrival handed the tick straight back to
> follow and formation movement, which pushed the bot off the staging point again — so it
> re-qualified, walked back, arrived, was pushed off, forever.

The arrival test then lives inside `Execute`, as a tolerance check that returns `false` without
moving:

```cpp
float const tolerance = isMelee ? 3.0f : 5.0f;
if (bot->GetDistance2d(moveX, moveY) < tolerance)
    return false;                       // in place: hold, cast, do not move
```
(`Aq40CthunMaintainSpreadAction::Execute`)

---

## M4. Anchor the boss at a fixed spot (tank job)

The standard tank-positioning action. Note the **step-limited** move: it moves at most 5y per
tick toward the anchor, dragging the boss rather than teleporting the tank across the room,
and passes `backwards = true` so the tank keeps facing the boss.

```cpp
if (najentus->GetVictim() == bot && bot->IsWithinMeleeRange(najentus))
{
    const float distToPosition = bot->GetExactDist2d(position);
    if (distToPosition > 3.0f)
    {
        const float moveDist = std::min(5.0f, distToPosition);
        const float moveX = bot->GetPositionX() + (dX / distToPosition) * moveDist;
        const float moveY = bot->GetPositionY() + (dY / distToPosition) * moveDist;
        return MoveTo(BLACK_TEMPLE_MAP_ID, moveX, moveY, bot->GetPositionZ(),
                      false, false, false, false, MovementPriority::MOVEMENT_COMBAT,
                      /*lessDelay*/ true, /*backwards*/ true);
    }
}
```
(`HighWarlordNajentusTanksPositionBossAction::Execute`)

The two guards matter: `GetVictim() == bot` means *this* tank actually has aggro, and
`IsWithinMeleeRange` means the boss will follow. Without them a tank with no aggro walks to the
anchor alone.

Positions live as `extern const Position` in `<X>Helpers.h`, defined in `<X>Helpers.cpp`. They
are surveyed in-game; annotate *why* a coordinate is what it is, as
`Aq40BossHelper::Twin::GetStagingPoint` does ("44.6y and 46.6y from them, 172 degrees between
the outbound legs… 78-81y from both spawns, comfortably outside aggro range").

---

## M5. Walk a bot through a multi-leg route

When a straight line to the anchor would path the boss through the raid or off a ledge, use a
per-bot step state machine keyed by `ObjectGuid`.

```cpp
enum class TankPositionState : uint8 { MovingToTransition, MovingToFinal, Positioned, Unknown };
extern std::unordered_map<ObjectGuid, TankPositionState> shahrazTankStep;
```

```cpp
auto it = shahrazTankStep.try_emplace(guid, TankPositionState::MovingToTransition).first;
const Position& position = it->second == TankPositionState::MovingToTransition
    ? SHAHRAZ_TRANSITION_POSITION : SHAHRAZ_TANK_POSITION;
// ... move; on arrival, advance the step
```
(`MotherShahrazTanksPositionBossUnderPillarAction::Execute`)

Other bots read the tank's progress via `GetShahrazTankPositionState()`, which looks up the
**main tank's** GUID — that is how "melee, wait at the safe spot until the tank is positioned"
is expressed without a second timer.

Same shape elsewhere: `tidewalkerTankStep` / `tidewalkerRangedStep` (SSC),
`nightbaneTankStep` / `nightbaneRangedStep` (Kara), `gathiosTankStep` / `zerevorHealStep` (BT).

**Waypoint-ring variant:** `flameTankWaypointIndex` (`std::unordered_map<ObjectGuid, size_t>`)
indexes into `E_GLAIVE_TANK_POSITIONS` / `W_GLAIVE_TANK_POSITIONS` (7 positions each) to walk
the Flames of Azzinoth tanks around a circuit.

These maps must be cleaned up — see [05-phases-timers-and-state.md](05-phases-timers-and-state.md#s4-cleaning-up).

---

## M6. Spread the raid — one deterministic slot each

Sort the eligible members by GUID, take your own index, convert to an angle. GUID sort is what
makes every bot agree on the assignment without any communication.

```cpp
std::sort(rangedBots.begin(), rangedBots.end(),
          [](Player* a, Player* b) { return a->GetGUID() < b->GetGUID(); });

int slotIndex = /* index of `bot` in rangedBots */;
float angleStep   = 2.0f * float(M_PI) / float(rangedBots.size());
float targetAngle = angleStep * float(slotIndex);
```
(`ICCActions_BPC.cpp`, Blood Prince Council — then nudges the angle in π/18 steps, up to 18
times, to dodge Shock Vortexes, so a slot that is unsafe degrades gracefully instead of
failing.)

**Roster stability is the whole game here.** If your eligibility filter includes something that
flickers (in-combat, in-range, has-aura), the roster size changes, every index shifts, and the
whole raid rotates. Filter on things that only change on death: role, class, alive.

Other slot schemes:
- `botAI->GetGroupSlotIndex(bot)` then `index % 2` / `index % 3` for coarse fan-out
  (`SapphironBossHelper::FindPosToAvoidChill`).
- `HealerAssignments::SideSplit()` — split a roster of any size into two sides, odd healer to
  the side that takes more damage (`Aq40BossHelper.h`).

---

## M7. Spread by distance rather than by slot

Cheaper, no roster needed, no determinism required. Push away from the nearest offender.

```cpp
constexpr float safeDistFromBoss = 10.0f;
if (bot->GetExactDist2d(najentus) < safeDistFromBoss &&
    FleePosition(najentus->GetPosition(), safeDistFromBoss, /*minInterval*/ 1000))
    return true;

constexpr float safeDistFromPlayer = 7.0f;
if (Player* nearestPlayer = GetNearestPlayerInRadius(bot, safeDistFromPlayer))
    return FleePosition(nearestPlayer->GetPosition(), safeDistFromPlayer, 1000);
```
(`HighWarlordNajentusDisperseRangedAction::Execute`)

`FleePosition(pos, radius, minInterval)` is a base-library primitive with its own rate limit —
use the `minInterval` rather than adding your own timer. `GetNearestPlayerInRadius` is in
`RaidBossHelpers.cpp` and only considers group members.

Used by `supremus disperse ranged`, `illidan stormrage disperse ranged`,
`illidari council disperse ranged`, `high warlord naj'entus disperse ranged`,
`azgalor disperse ranged`, `rs saviana melee spread`.

---

## M8. Find a safe spot, with a safe path to it

Two implementations, same idea: sample a polar grid around the bot, reject positions inside a
hazard, then *also* reject positions whose straight-line path crosses a hazard.

```cpp
for (float distance = 0.0f; distance <= maxRadius; distance += 1.0f)
    for (float angle = 0.0f; angle < 2*M_PI; angle += M_PI/8.0f)
    {
        // reject if within hazardRadius of any hazard
        // prefer positions where IsPathSafeFromVolcanos(bot, testPos, ...) too
    }
```
(`SupremusMoveAwayFromVolcanosAction::FindSafestNearbyPosition` / `::IsPathSafeFromVolcanos` —
path check samples 10 points along the segment.)

The generalised versions to prefer:
- `KarazhanHelpers::TryFindSafePositionWithSafePath(...)` — parameterised angles, step size,
  max sample distance, and a `requireSafePath` flag.
- `ZulAmanHelpers::FindSafestNearbyPosition(...)` / `IsPathSafeFromHazards` / `IsPositionSafeFromHazards`.

**Nearest-safe-position, no search:** `CombatFormationMoveAction::GetNearestPosition(const std::vector<Position>&)`
when you have a fixed candidate list rather than a continuous space.

**Floor validity:** AQ40 wraps every C'Thun destination in `ResolveSafeCthunFloorPosition`
before moving, and logs `movement_failure … no_safe_floor` when it cannot find one. If your
encounter has holes in the floor, do the same — `MoveTo` will happily aim at nothing.

---

## M9. Orbit / rotate around a point

**Step around the boss, away from a beam.** Pick the rotation direction from the sign of the
cross product between the beam direction and the bot's bearing, so the bot always turns the
short way out:

```cpp
float cross = beamDx * botDy - beamDy * botDx;
float nextAngle = currentAngle + (cross >= 0.0f ? step : -step);   // step = π/5
float moveX = boss->GetPositionX() + std::cos(nextAngle) * radius;
```
(`Aq40CthunAvoidDarkGlareAction::Execute`. `VezaxShadowCrashAction` does the same with a fixed
π/10 step and no direction logic.)

**Fixed waypoint ring:** `RotateAroundTheCenterPointAction` (base library) precomputes N
waypoints on a circle in its constructor and exposes `FindNearestWaypoint()`. Use this when the
ring is a property of the room rather than of the boss.

---

## M10. Kite

```cpp
constexpr float safeDistance = 25.0f;
const float currentDistance = bot->GetDistance2d(supremus);
if (currentDistance < safeDistance)
    return MoveAway(supremus, safeDistance - currentDistance);
```
(`SupremusKiteBossAction::Execute` — note it moves the *deficit*, not the full distance.)

Others: `kael'thas sunstrider kite thaladred`, `icc valithria zombie kite`,
`lady vashj tank attack and move away strider`, `mc move from baron geddon`.

---

## M11. Stand behind / avoid the frontal

Do not roll your own arc maths. `RearFlankAction` (base library) already encodes the reasoning:

> 90 degree minimum angle prevents any frontal cleaves/breaths and avoids parry-hasting the
> boss. 120 degree maximum angle leaves a 120 degree symmetrical cone at the tail end which is
> usually enough to avoid tail swipes.

```cpp
RearFlankAction(botAI, distance, ANGLE_90_DEG, ANGLE_120_DEG)
```

Wired directly from a strategy where it is enough:
`NextAction("rear flank", ACTION_RAID + 1)` (Naxx, Maexxna). Boss-specific variants exist
(`grobbulus go behind the boss`, `the lurker below run around behind boss`,
`rs baltharus avoid front`, `rs saviana avoid front`) where the safe arc differs.

---

## M12. Stage the raid before a pull

```cpp
bool ShouldStageForPull(Player* bot, PlayerbotAI* botAI)
{
    if (!bot->IsAlive() || bot->IsInCombat() || !IsInAq40(bot))       return false;
    if (distance_to_staging > kStagingApplyRange /*120y*/)            return false;  // outer gate only
    if (either emperor IsAlive() && IsInCombat())                     return false;  // belt
    if (either emperor IsEngagedWithRaid(...))                        return false;  // and braces
    return true;
}
```
(`Aq40BossHelper::Twin::ShouldStageForPull`)

Four things to copy:

1. **Outer range gate only, no inner cut-off** — see M3.
2. **Shared between the trigger and the action**, so the two can never disagree about whether
   staging is live.
3. **Two independent "encounter started" tests.** `IsEngagedWithRaid` needs the boss to have a
   player victim, which is a live read that can blink empty. An action that walks the raid into
   the middle of the room must not fire on a blink, so the raw combat flag is checked first.
4. **Apply range large enough to cover the whole approach** (120y here) but small enough that a
   bot elsewhere in the instance does not set off walking.

---

## Movement primitive reference

From [MovementActions.h](../../src/Ai/Base/Actions/MovementActions.h):

| Call | Use |
|---|---|
| `MoveTo(mapId, x, y, z, idle, react, normal_only, exact_waypoint, priority, lessDelay, backwards)` | the workhorse |
| `MoveTo(WorldObject*, distance, priority)` | move to within `distance` of a thing |
| `MoveNear(target, distance, priority)` | |
| `MoveInside(mapId, x, y, z, distance, priority)` | get within a radius, do not overshoot |
| `MoveAway(target, distance, backwards)` | |
| `FleePosition(pos, radius, minInterval)` | rate-limited push-away |
| `JumpTo(mapId, x, y, z, priority)` | used to drop between Thaddius platforms |
| `MoveToLOS(target, ranged)` | |
| `Move(angle, distance)` / `MoveFromGroup(distance)` | |

`MovementPriority`: `MOVEMENT_IDLE` < `MOVEMENT_WANDER` < `MOVEMENT_NORMAL` < `MOVEMENT_COMBAT`
< `MOVEMENT_FORCED`. Higher overrides a lower in-flight move. Encounter positioning uses
`MOVEMENT_COMBAT`; "you will die if you do not move" uses `MOVEMENT_FORCED`.

Beware `IsWaitingForLastMove(priority)`: a move already in flight at equal-or-higher priority
makes your `MoveTo` a no-op and it returns `false`. This is the cause of the classic
"`moved=0` in the log while the bot sails across the room" symptom — see
[08-gotchas.md](08-gotchas.md#g3-your-movetomovenear-silently-did-nothing).
