# 04 — Hazards and avoidance

A hazard is one of five things in this codebase, and which one it is determines how you find it.
Getting this wrong is why an avoidance action "does nothing" — you searched for a creature and
the hazard is a dynamic object.

| Hazard is… | Find it with |
|---|---|
| a persistent-area aura (Blizzard, Consecration, Death and Decay) | `GetDynamicObjectPositions(bot, radius, spellId)` |
| a spawned creature (volcano, fire bomb, void zone, vortex) | `bot->GetCreatureListWithEntryInGrid(list, entry, radius)` |
| a trigger NPC | `AI_VALUE(GuidVector, "nearest triggers")` |
| a gameobject | `AI_VALUE(GuidVector, "nearest game objects")` / `bot->FindNearestGameObject(entry, radius, true)` |
| a debuff on a player, with no world object at all | `member->HasAura(spellId)` |

Plus a sixth, for things that have not landed yet: a server-side spell hook that records the
impact point. See [06-server-side-hooks-and-cheats.md](06-server-side-hooks-and-cheats.md).

---

## H1. Dynamic-object AoE

`RaidBossHelpers.cpp` does the grid search for you:

```cpp
std::vector<Position> GetDynamicObjectPositions(Player* bot, float searchRadius, uint32 spellId);
```

It visits `GRID_MAP_TYPE_MASK_DYNAMICOBJECT` and filters on `dynObj->GetSpellId() == spellId`.

The generic `AvoidAoeAction` (base library) covers the common case without any encounter code
at all, and several strategies simply wire it up:

```cpp
triggers.push_back(new TriggerNode("faerlina", { NextAction("avoid aoe", ACTION_RAID + 1) }));
```
(`NaxxStrategy.cpp`)

Its three internal probes are `AvoidAuraWithDynamicObj()`, `AvoidGameObjectWithDamage()` and
`AvoidUnitWithDamageAura()`. **Try `avoid aoe` before writing anything.**

> ⚠️ `avoid aoe` has relevance ~90 and does not know that standing in the fire is sometimes the
> job. See [08-gotchas.md](08-gotchas.md#g4-avoid-aoe-will-walk-your-tank-out-of-the-boss).

---

## H2. Creature hazards

```cpp
std::list<Creature*> creatureList;
bot->GetCreatureListWithEntryInGrid(creatureList, NPC_SUPREMUS_VOLCANO, /*radius*/ 40.0f);
for (Creature* c : creatureList)
    if (c && c->IsAlive())
        hazards.push_back(c);
```
(`SupremusMoveAwayFromVolcanosAction::GetAllSupremusVolcanos`; the same helper exists as
`ZulAmanHelpers::GetAllHazardTriggers(bot, entry, radius)` and
`KarazhanHelpers::GetAllVoidZones(botAI, bot)`.)

Then: in-danger test → safe-position search → move. See
[01-movement-and-positioning.md](01-movement-and-positioning.md#m8-find-a-safe-spot-with-a-safe-path-to-it)
for the search.

The reusable no-code version is `MoveAwayFromCreatureAction(botAI, name, creatureId, range, alive)`
from the base library — construct it in your `ActionContext` and you are done:

```cpp
static Action* some_avoid(PlayerbotAI* ai) { return new MoveAwayFromCreatureAction(ai, "…", NPC_X, 15.0f); }
```

---

## H3. Cones, beams and swept lines

**Cone / facing test** — cheap, when the hazard is "in front of the boss":

```cpp
if (!isEncounterTank && ouro->isInFront(bot, 10.0f) && bot->GetDistance2d(ouro) <= 15.0f)
    // Sand Blast window
```
(`Aq40OuroMultiplier::GetValue`)

**Rotating beam** — step around the boss, turning the short way out. The cross product picks
the direction:

```cpp
float beamDx = std::cos(boss->GetOrientation()), beamDy = std::sin(boss->GetOrientation());
float botDx = dx / radius,                       botDy = dy / radius;
float cross = beamDx * botDy - beamDy * botDx;
float nextAngle = currentAngle + (cross >= 0.0f ? step : -step);
```
(`Aq40CthunAvoidDarkGlareAction::Execute`)

**Swept line / capsule** — the general one, for a hazard travelling from A to B with a width.
`BTHelpers.cpp` has both halves, and this is the piece worth stealing:

```cpp
struct EyeBlastDangerArea { Position start; Position end; float width; };

EyeBlastDangerArea GetEyeBlastDangerArea(Player* bot);   // start = trigger NPC position,
                                                         // end   = trigger->GetMotionMaster()->GetDestination()
bool IsPositionInEyeBlastDangerArea(const Position& pos, const EyeBlastDangerArea& area);
```

`IsPositionInEyeBlastDangerArea` is a clamped point-to-segment distance test — project onto the
segment, clamp the parameter to [0,1], measure. **Reading the trigger NPC's MotionMaster
destination to learn where a moving hazard is going** is the transferable trick here; it works
for any hazard implemented as a moving trigger creature.

Also: `rs halion avoid cones`, `rs halion p2 avoid cones`, `rs baltharus avoid front`,
`rs saviana avoid front`, `mimiron p3wx2 laser barrage`, `kologarn eyebeam`.

---

## H4. React to a debuff on yourself

```cpp
uint32 idx;
if (NaxxSpellIds::HasAnyAura(bot, {NegativeCharge10, NegativeCharge25, NegativeChargeStack}) ||
    botAI->HasAura("negative charge", bot, false, false, -1, true))              idx = 0;
else if (/* positive */)                                                          idx = 1;
else                                                                              idx = 2;   // uncharged

idx = idx * 2 + botAI->IsRanged(bot);      // 6 positions: {left,right,centre} x {melee,ranged}
return MoveTo(mapId, position[idx].first, position[idx].second, bot->GetPositionZ(), …);
```
(`ThaddiusMovePolarityAction::Execute`)

Two things to copy: the **ID-then-name fallback** (see below), and the **third "no debuff yet"
case** — without it, bots between polarity applications have no assignment and drift.

Reusable version: `MoveAwayFromPlayerWithDebuffAction(botAI, name, spellId, range)`.

### Spell-ID lookup with a name fallback

Every mature instance has a `<X>SpellIds.h` with helpers of this shape, and every call site
falls back to the name because custom/patched spell data may renumber:

```cpp
NaxxSpellIds::HasAnyAura(bot, {Chill10, Chill25})
    || botAI->HasAura("chill", bot);                       // fallback for custom spell data

Aq40SpellIds::MatchesAnySpellId(spell->GetSpellInfo(), {SarturaWhirlwind, SarturaGuardWhirlwind})
    || botAI->HasAura("whirlwind", unit);
```

Do both. ID-only breaks on non-Blizzlike data; name-only breaks on localisation and is slow.

---

## H5. React to a debuff on someone else

Scan the group, then decide whether *you* are the one who should act — see
[02-coordination-and-assignment.md](02-coordination-and-assignment.md#c9-picking-the-one-bot-nearest-to-something).

```cpp
Player* enragedPlayer = /* group member with SPELL_PLAYER_FEL_RAGE */;
constexpr float safeDistance = 20.0f;
if (enragedPlayer && bot->GetExactDist2d(enragedPlayer) < safeDistance)
    return FleePosition(enragedPlayer->GetPosition(), safeDistance, /*minInterval*/ 0);
```
(`GurtoggBloodboilRangedMoveAwayFromEnragedPlayerAction::Execute`)

Related: `mother shahraz run away to break fatal attraction` (two linked players must separate),
`icc bql pact of darkfallen`, `illidan stormrage isolate bot with parasite`,
`hex lord malacrass move away from freezing trap`, `lady vashj static charge move away from group`.

---

## H6. Read a cast in progress

```cpp
Spell* spell = unit->GetCurrentSpell(CURRENT_GENERIC_SPELL);
if (!spell) spell = unit->GetCurrentSpell(CURRENT_CHANNELED_SPELL);   // channels are a different slot
if (spell && Aq40SpellIds::MatchesAnySpellId(spell->GetSpellInfo(), { … })) …
```

Check **both** slots. `GluthBossHelper::BeforeDecimate` and `IsSarturaSpinning` do; code that
only checks `CURRENT_GENERIC_SPELL` will miss every channel.

Belt-and-braces version (cast **or** resulting aura **or** name), because the cast window is
short and you may arrive mid-effect:

```cpp
bool IsSarturaSpinning(PlayerbotAI* botAI, Unit* unit)
{
    Spell* spell = unit->GetCurrentSpell(CURRENT_GENERIC_SPELL);
    return (spell && MatchesAnySpellId(spell->GetSpellInfo(), {Whirlwind, GuardWhirlwind}))
        || HasAnyAura(botAI, unit, {Whirlwind, GuardWhirlwind})
        || botAI->HasAura("whirlwind", unit);
}
```

---

## H7. Prioritising the avoidance over everything else

Avoidance actions are wired at `ACTION_EMERGENCY + n`, and the multiplier boosts them above 1.0
while the danger window is open, then zeroes everything that would fight them:

```cpp
if (dynamic_cast<Aq40BugTrioAvoidPoisonCloudAction*>(action))     return 3.5f;
if (IsGenericMovementAction(action) || IsAttackOrReachAction(action)) return 0.0f;
```
(`Aq40BugTrioMultiplier::GetValue`)

Observed AQ40 boost scale, which is a reasonable house convention:

| Multiplier | Meaning |
|---|---|
| `4.0f` | you will die — Dark Glare, Twin hazard, Plague AoE, stomach exit at stack threshold |
| `3.5f` | serious — poison cloud, whirlwind, sand blast, Vek'lor proximity |
| `3.0f` | job-critical — tank stations, healer anchor, stomach DPS, vulnerable burst |
| `2.0f–2.8f` | encounter target selection, Viscidus shatter |
| `0.4f–0.5f` | discouraged but allowed |
| `0.0f` | forbidden |

The exemption list is the fiddly part: your avoidance action, `"avoid aoe"`, and any other
avoidance that must still run, all have to be excluded from the `0.0f`:

```cpp
if (IsGenericMovementAction(action) &&
    !IsActionNamed(action, {"aq40 twin avoid hazard", "aq40 twin avoid veklor", "avoid aoe"}))
    return 0.0f;
```
