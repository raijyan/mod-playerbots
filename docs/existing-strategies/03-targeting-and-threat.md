# 03 — Targeting, threat and kill order

The bots have a generic targeting engine (`DpsAssistAction`, `TankAssistAction`,
`AttackRtiTargetAction`, `ChooseTargetActions`). Encounter logic overrides it in two steps that
must **both** happen:

1. an action that picks the right target, at `ACTION_RAID + n`;
2. a multiplier that zeroes the generic targeting so it does not immediately pick something else.

Doing only (1) produces a bot that flickers between two targets every tick. That is the single
most common targeting bug in this tree.

---

## T1. Override the target

The convention is an action named `<instance|boss> choose target`. There are 12 of them
(`aq40 cthun choose target`, `gluth choose target`, `kel'thuzad choose target`,
`anub'rekhan choose target`, `loatheb choose target`, `aq40 sartura choose target`, …).

Shape:

```cpp
Unit* target = /* encounter-specific selection */;
if (!target)
    return false;

// idempotent: if we are already on it, do not re-issue and do not consume the tick
if (AI_VALUE(Unit*, "current target") == target && bot->GetVictim() == target)
    return false;

return Attack(target);
```
(`Aq40CthunChooseTargetAction::Execute`)

Both halves of the idempotence test matter. `current target` is the AI's intent; `GetVictim()`
is the server's reality. They diverge for a tick after `Attack()`, and testing only one gives
you a re-issue every tick.

**Pinning.** AQ40 additionally writes the choice into the context so other systems see it —
`PinCthunTarget(botAI, context, target)`. Use this when generic actions downstream read
`"current target"` and you need them to agree.

---

## T2. Enforce a kill order

```cpp
std::vector<Unit*> attack_order;
if (botAI->IsAssistTank(bot)) attack_order = {fourth, thane, lady, sir};
else                          attack_order = {thane, fourth, lady, sir};

for (Unit* t : attack_order)
    if (t && t->IsAlive()) { target = t; break; }
```
(`FourHorsemenAttackInOrderAction::Execute` — note the order differs by role.)

Variants: `<boss> assign dps priority` (Karathress, Halazzi, Hex Lord, Maulgar, Magtheridon,
Illidari Council, Leotheras, Vashj, Kael'thas) — same idea with role-conditioned lists.
`aq40 cthun phase2 add priority`, `illidan stormrage dps prioritize adds`.

Do not forget the LoS fallback, or bots behind a pillar will stand still holding a target they
cannot hit:

```cpp
if (!bot->IsWithinLOSInMap(target))
    return MoveNear(target, 22.0f, MovementPriority::MOVEMENT_COMBAT);
return Attack(target);
```

---

## T3. Suppress the generic targeting engine

In the multiplier, kill the generic assist actions and any competing encounter action:

```cpp
if (StartsWith(actionName, "aq40 trash "))                                  return 0.0f;
if (dynamic_cast<DpsAssistAction*>(action) || dynamic_cast<TankAssistAction*>(action))
                                                                            return 0.0f;
```
(`Aq40SkeramMultiplier::GetValue`, `Aq40FankrissMultiplier::GetValue`)

Or, where the whole instance-level target chooser must stand down for one boss:

```cpp
if (actionName == "aq40 choose target")   return 0.0f;
```
(`Aq40OuroMultiplier`, `Aq40ViscidusMultiplier`, `Aq40CthunMultiplier`)

The broader "no pressure at all" predicate:

```cpp
bool IsGenericPressureAction(Action* action)
{
    return IsAttackOrReachAction(action) ||        // AttackAction | ReachTargetAction | CastReachTargetSpellAction
           IsOffensiveSpellAction(action) ||       // CastSpellAction that is not CastHealingSpellAction
           dynamic_cast<PetAttackAction*>(action) ||
           action->getName() == "shoot";
}
```
(`Aq40Multipliers.cpp` — note the pet and autoshoot cases, which are easy to forget and which
are exactly what keeps a "stopped" bot in combat.)

**Pets need separate handling.** Suppressing the owner's actions does not stop the pet:

```cpp
if (!isWarlockTank && PetIsTargetingVeklor(bot) &&
    (dynamic_cast<PetAttackAction*>(action) || dynamic_cast<SetPetStanceAction*>(action) ||
     dynamic_cast<TogglePetSpellAutoCastAction*>(action)))
    return 0.0f;
```
See also `illidan stormrage control pet aggression`, `nightbane control pet aggression`,
`illidari council command pets to attack gathios`.

---

## T4. Do not pull before the tank has it

```cpp
bool const isTankControlledAdd = target && (name == "giant claw tentacle" || name == "claw tentacle");
if (isTankControlledAdd && Aq40BossHelper::ShouldWaitForEncounterTankAggro(bot, bot, target))
    return false;
```
(`Aq40CthunChooseTargetAction::Execute`; see
[02-coordination-and-assignment.md](02-coordination-and-assignment.md#c8-wait-for-the-tank-without-a-timer))

---

## T5. Hold DPS for N seconds

When there is nothing observable to wait on — an aggro reset on a phase change, say — use an
instance-keyed timer stamped by a `manage … timer` action, and read it in a multiplier.

Stamp (an `ACTION_EMERGENCY + 10` action that returns `false`, so it never costs a tick):

```cpp
supremusPhaseTimer.try_emplace(supremus->GetMap()->GetInstanceId(), std::time(nullptr));
return false;
```

Read:

```cpp
const time_t now = std::time(nullptr);
constexpr uint8 dpsWaitSeconds = 5;
auto it = councilDpsWaitTimer.find(gathios->GetMap()->GetInstanceId());
if (it == councilDpsWaitTimer.end() || (now - it->second) >= dpsWaitSeconds)
    return 1.0f;                                    // window over: normal

if (dynamic_cast<AttackAction*>(action) ||
    (dynamic_cast<CastSpellAction*>(action) && !dynamic_cast<CastHealingSpellAction*>(action)))
    return 0.0f;                                    // in window: DPS holds, healing continues
return 1.0f;
```
(`IllidariCouncilWaitForDpsMultiplier::GetValue`)

Three details to copy: **tanks are exempted at the top** of the function; **the misdirect action
is exempted** (it is what the wait is *for*); and **healing is not suppressed**, because
`CastHealingSpellAction` is excluded from the cast test.

`try_emplace` not `operator[]` — the point is to stamp the *first* time, not every tick.

Instances with this: BT (`illidanBossDpsWaitTimer`, `illidanFlameDpsWaitTimer`,
`councilDpsWaitTimer`), SSC (`hydrossFrostDpsWaitTimer`, `hydrossNatureDpsWaitTimer`,
`karathressDpsWaitTimer`, `leotheras*DpsWaitTimer` ×3), Kara (`attumenDpsWaitTimer`,
`netherspiteDpsWaitTimer`, `nightbaneDpsWaitTimer`).

---

## T6. Hold burst cooldowns until the pull has settled

```cpp
float SupremusDelayDpsCooldownsMultiplier::GetValue(Action* action)
{
    Unit* supremus = AI_VALUE2(Unit*, "find target", "supremus");
    if (!supremus || supremus->GetHealthPct() < 95.0f)
        return 1.0f;                                  // already engaged: burst freely

    if (IsDpsCooldownAction(action) || (botAI->IsDps(bot) && dynamic_cast<UseTrinketAction*>(action)))
        return 0.0f;
    return 1.0f;
}
```

`IsDpsCooldownAction` is a static free function at the top of `BTMultipliers.cpp` listing 22
cooldown actions (Heroism, Bloodlust, Icy Veins, Arcane Power, Avenging Wrath, Army of the
Dead, Berserking, Blood Fury, …). **Reuse it rather than re-listing** — a new class cooldown
should be added there once, not in every instance. Nearly every BT/SSC/Kara/ZA/Hyjal boss has a
`<Boss>DelayDpsCooldownsMultiplier` that is otherwise identical.

---

## T7. Deciding "the encounter has actually started"

Do **not** use `unit->GetTarget()`. It is set on a boss idling in its own room long before
anyone pulls it. The comment in `Aq40BossHelper.h` is worth quoting in full because the bug it
describes is expensive:

> The old test accepted a bare `unit->GetTarget()`, which is true of a boss standing idle in its
> own room, so every per-boss "\<boss\> active" trigger read as live while the raid was still on
> the trash — and those triggers drive AttackActions, so the raid pulled the boss on sight.

Use:

```cpp
bool IsEngagedWithRaid(Player* bot, PlayerbotAI*, Unit* unit)
{
    if (!unit->IsInCombat())                       return false;
    Unit* victim = unit->GetVictim();
    if (!victim)                                   return false;
    Player* victimPlayer = victim->ToPlayer();
    if (!victimPlayer)
        if (Unit* owner = victim->GetOwner())      // a pet counts as us
            victimPlayer = owner->ToPlayer();
    return victimPlayer && IsSameInstance(bot, victimPlayer);
}
```

And note it can blink empty for a tick, so anything expensive (walking the raid across a room)
should also check the raw combat flag — see
[01-movement-and-positioning.md](01-movement-and-positioning.md#m12-stage-the-raid-before-a-pull).

Two related sets in the same header, which are deliberately different:

- `GetEncounterUnits(...)` — awareness. Anything hostile and nearby, LoS or not.
- `GetActiveCombatUnits(...)` — engagement. Only things actually fighting us.

Use `EncounterUnits` to *find* the boss and `ActiveCombatUnits` to decide whether the encounter
is live. Confusing the two is what caused the bug above.

---

## T8. Threat-model helpers you may not know exist

```cpp
AI_VALUE2(bool, "has aggro", "current target")             // do I have threat
AI_VALUE2(bool, "has aggro", "boss target")
IsUnitFocusedOnPlayer(unit, player)                        // unit->GetVictim() == player || unit->GetTarget() == guid
IsUnitHeldByEncounterTank(referencePlayer, unit, primaryOnly)
HasAnyNamedUnitHeldByEncounterTank(botAI, ref, units, {names}, primaryOnly)
```

For taunt/tank-swap: the encounter action is normally just
`NextAction("taunt spell", ACTION_RAID + n)` wired to a debuff-stack trigger
(`gluth main tank mortal wound`, `thaddius phase pet lose aggro`). Do not write a bespoke taunt
action; write the trigger.

Explicit swap actions exist where the swap is more than a taunt:
`aq40 fankriss tank swap`, `al'ar swap tanks on boss`, `bwl ebonroc taunt`.
