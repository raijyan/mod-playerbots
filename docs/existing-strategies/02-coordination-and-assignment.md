# 02 — Coordination and assignment

Bots do not talk to each other. Every "the raid agrees on X" mechanism in this tree works by
each bot independently computing the *same* answer from shared world state. That constraint
shapes everything below: the answer must be **deterministic** and **stable**, or the raid
oscillates.

---

## C1. Exactly one bot does a raid-wide job

The job is "who owns the raid markers / who maintains the shared map / who does the thing that
must happen once". Election is by first-alive-in-group-order:

```cpp
bool IsMechanicTrackerBot(Player* bot, uint32 mapId)   // RaidBossHelpers.cpp
{
    for (GroupReference* ref = group->GetFirstMember(); ref; ref = ref->next())
    {
        Player* member = ref->GetSource();
        if (!member || !member->IsAlive() || member->GetMapId() != mapId || !GET_PLAYERBOT_AI(member))
            continue;
        return member == bot;      // first survivor of the filter wins
    }
    return false;
}
```

Use it in a **trigger**, not an action, so only the elected bot's action is ever queued:

```cpp
bool TheCuratorMarkAstralFlareTrigger::IsActive()
{
    return IsMechanicTrackerBot(bot, KARAZHAN_MAP_ID) && /* … */;
}
```

Users: Kara (10 sites), BT (5), SSC (5), TK (2), ZA, Mag.

**Properties worth knowing.** The winner changes when the current winner dies or leaves the
map — that is deliberate, it is a liveness property, and it means the job is never orphaned.
It also means the job is *not* sticky: do not use this for anything where a handover mid-job is
harmful (a multi-tick sequence, a partially-passed item). It skips real players
(`!GET_PLAYERBOT_AI` filters them out), so a human in the raid never gets elected.

---

## C2. Raid target icons

`RaidBossHelpers.h` has the whole family:

```cpp
MarkTargetWithSkull(bot, target);      // and Square/Star/Circle/Diamond/Triangle/Cross/Moon
MarkTargetWithIcon(bot, target, iconId);
ClearTargetIcon(bot, iconId);
```

They are idempotent — `MarkTargetWithIcon` returns `false` (no-op) if that icon is already on
that target, so calling every tick is safe and cheap.

> The header notes: *"these functions do not allow the player to change the icon during the
> encounter"* — a bot re-asserts its icon every tick, so a human's manual re-mark is undone.

**You must clear what you set.** An icon is not decoration to a bot raid:
`AttackRtiTargetAction` sends every non-healer at whatever wears the skull, **and it does not
require combat to do so**. A skull left on a boss after a wipe means the raid walks back up the
ramp and re-pulls it on sight. That bug was real; the fix is
`Aq40Helpers::ClearStaleEncounterMarkers`, which only clears icons that are on an encounter
unit *not currently fighting us*, so a marker a human placed on something else is left alone.

The logging-aware variants `Aq40Helpers::SetRaidTargetIcon(bot, target, iconId, boss, marker)`
and `ClearRaidTargetIcon(...)` verify the write took effect and emit a `raid_marker` log line.
Copy those if you are debugging marker behaviour.

**RTI vs icon.** Two different things:
- **Group target icon** — the world-visible skull. `group->SetTargetIcon(...)`.
- **`rti` / `rti target` context values** — this bot's *own* notion of the assist target.
  `SetRtiTarget(botAI, "skull", target)` sets both.

---

## C3. Finding the tanks

Two tiers.

**Simple (`RaidBossHelpers.h`)** — good enough for most encounters:

```cpp
Player* GetGroupMainTank(PlayerbotAI* botAI, Player* bot);          // requires the MT alive
Player* GetGroupAssistTank(PlayerbotAI* botAI, Player* bot, uint8 index);
```
`GetGroupAssistTank` prefers group *assistants* first, then falls back to any other tank, so a
raid that has not been set up properly still resolves an index.

**Rich (`Aq40BossHelper.h`)** — a cached snapshot with a documented precedence chain:

```cpp
Player* GetEncounterPrimaryTank(Player* player);
Player* GetEncounterBackupTank(Player* player, uint8 index = 0);
bool    IsEncounterTank(Player* referencePlayer, Player* player);
```

Precedence for primary: `IsExplicitMainTank` → `IsTank && IsMainTank` → any tank.
Backups are stable-sorted by `IsAssistTankOfIndex(0)` → `(1)` → `IsAssistTank` →
`group->IsAssistant` → GUID, so the order does not flicker.

The snapshot (`Detail::EncounterMemberSnapshot`) is `thread_local` and invalidated on
`getMSTime()` change / bot change / map change, so it costs one group walk per bot per
millisecond rather than one per call. Copy this if you are calling tank lookup from a
multiplier (which runs once per candidate action, i.e. dozens of times a tick).

---

## C4. Assigning N bots to N distinct jobs

The pattern is always: **collect candidates → stable-sort → take your index**.

`Aq40BossHelper::Twin::GetTankPairAssignments` is the fullest example. It picks two warlock
tanks and two melee tanks, and note the three things it does that a naive version does not:

1. **Two candidate pools, preferred and fallback.** Group assistants first, then any bot.
   ```cpp
   if (group && group->IsAssistant(member->GetGUID())) AppendUniquePlayer(assistantWarlocks, member);
   else if (GET_PLAYERBOT_AI(member))                  AppendUniquePlayer(fallbackWarlocks, member);
   ```
2. **A priority function, then GUID as the tiebreak** — `GetTwinMeleeTankCandidatePriority`
   returns 0..5 (explicit MT, runtime MT, assistant tank, tank, role-tank assistant, role-tank)
   and `stable_sort` breaks ties by `GetGUID().GetRawValue()`.
3. **An explicit degradation mode**, not a silent partial result:
   ```cpp
   enum class TankAssignmentMode : uint8 { Incomplete, SinglePairFallback, FullPairs };
   ```
   Callers ask `HasFullPairs()` / `HasSinglePair()` / `HasUsableAssignment()` and behave
   differently, instead of dereferencing a half-filled array.

The same file's `GetIllidanWarlockTank` / `GetZerevorMageTank` (BT) are the two-line version of
the same idea when you need exactly one:

```cpp
// (1) First priority is an assistant Warlock (real player or bot)
// (2) If no assistant Warlock, then look for any Warlock bot
```

---

## C5. Splitting a roster into sides

`Aq40BossHelper::Twin::HealerAssignments`. The comment records the mistake that produced it:

> Every healer in the instance group, not a fixed four. A raid brings as many as it likes — 12
> in Mike's — and capping the roster at four left the other eight with no side, no anchor and
> no duty, healing whoever happened to be near them while a tank died.

```cpp
uint8 SideSplit() const { return uint8((healers.size() + 1) / 2); }   // odd healer to the heavier side
bool  IsSkullSideSlot(uint8 slot) const { return slot < SideSplit(); }
```

**Never hard-code a roster size.** Derive the split from `size()`.

---

## C6. Rotating the raid through a stacking debuff

Bloodboil-style: N groups, one takes the hit at a time, on a clock.

```cpp
// build: ranged members, 5 per group, 3 groups
for (size_t i = 0; i < rangedMembers.size(); ++i)
{
    groups[i / 5].push_back(rangedMembers[i]);
    if (groups[2].size() == 5) break;
}

// select: 3 groups, swapping every 10 seconds
const int groupIndex = (elapsed % 30) / 10;
```
(`GetGurtoggRangedRotationGroups` / `GetGurtoggActiveRotationGroup`, `BTHelpers.cpp`)

Then each bot asks only "am I in the active group?" and moves to the far or near position
accordingly (`GurtoggBloodboilRotateRangedGroupsAction`). No coordination needed — the clock is
shared because `gurtoggPhaseTimer` is keyed by instance id.

Note this one does **not** GUID-sort, so the grouping depends on group iteration order. That is
acceptable here because membership only needs to be *consistent within a tick*, not stable
across the fight — but it is a weaker guarantee than C4, and if you copy it and then need
stability you will have to add the sort.

---

## C7. Chaining hunter misdirects to different tanks

The recurring "pull" mechanic — 20 bosses have a `misdirect boss to main tank` action.

Single-tank version:

```cpp
Player* mainTank = GetGroupMainTank(botAI, bot);
if (botAI->CanCastSpell("misdirection", mainTank))
    return botAI->CastSpell("misdirection", mainTank);

// then, once the buff is up, actually shoot
if (bot->HasAura(SPELL_MISDIRECTION) && botAI->CanCastSpell("steady shot", boss))
    return botAI->CastSpell("steady shot", boss);
```
(`HighWarlordNajentusMisdirectBossToMainTankAction::Execute`)

Multi-target version — first three hunters in group order map to MT / assist 0 / assist 1:

```cpp
std::vector<Player*> hunters;   // first 3 alive hunters, group order
if      (bot == hunters[0] && mainTank)                          misdirectTarget = mainTank;
else if (hunters.size() > 1 && bot == hunters[1] && assist0)     misdirectTarget = assist0;
else if (hunters.size() > 2 && bot == hunters[2] && assist1)     misdirectTarget = assist1;
```
(`SupremusMisdirectBossToMainTankAction::Execute`)

The trigger is normally `bot->getClass() == CLASS_HUNTER && boss->GetHealthPct() > 95.0f`
(i.e. "not pulled yet"), at `ACTION_RAID + 2` so it beats ordinary positioning.

---

## C8. "Wait for the tank" without a timer

Rather than a timer, read the tank's state directly.

**Aggro-based** — the general one, in `Aq40BossHelper.h`:

```cpp
bool ShouldWaitForEncounterTankAggro(Player* referencePlayer, Player* player, Unit* unit, bool primaryOnly);
// false if you ARE a tank, false if the unit is already on you,
// false if there is no assigned tank at all (otherwise nobody would ever attack),
// otherwise: true unless the unit is focused on one of the encounter tanks.
```

That `!hasAssignedTank → false` branch is important. Without it, a raid with no recognised tank
would stand and watch.

**Step-based** — read the tank's positioning state machine (see
[01-movement-and-positioning.md](01-movement-and-positioning.md#m5-walk-a-bot-through-a-multi-leg-route)):

```cpp
GetShahrazTankPositionState(botAI, bot) != TankPositionState::Positioned   // → melee wait at the safe spot
```

**Timer-based** — only when there is genuinely nothing to observe. See
[03-targeting-and-threat.md](03-targeting-and-threat.md#t5-hold-dps-for-n-seconds).

---

## C9. Picking the one bot nearest to something

Where the job belongs to whoever is closest rather than to a fixed owner, compute it inside the
**trigger** and have it return true for exactly one bot:

```cpp
// find the impaled player, then find the closest non-tank bot to them,
// then: return closestBot == bot;
```
(`HighWarlordNajentusPlayerIsImpaledTrigger::IsActive`)

Every bot runs the same scan and only one gets `true`. This is the "nearest wins" analogue of
C1, and it is stable as long as nobody is moving fast — if two bots are near-equidistant it can
flap, so pair it with a job that is idempotent (walking to a spine) rather than one that is not
(consuming an item).

`GetNearestPlayerInRadius(bot, radius)` in `RaidBossHelpers.cpp` is the group-scoped nearest
lookup.
