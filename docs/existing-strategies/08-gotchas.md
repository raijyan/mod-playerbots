# 08 — Gotchas

Every entry here is a bug that already happened and cost real debugging time. They are ordered
roughly by how often they bite.

---

## G1. Your formation will not hold, because follow runs in a separate pass

**Symptom.** The whole raid runs back and forth forever. Bots reach their spread positions,
immediately walk off, walk back, walk off.

**Cause.** Your positioning action returns `false` on arrival (deliberately, so the bot can
cast). That hands the tick to `FollowAction` / `CombatFormationMoveAction`, which move it off.
It then re-qualifies for your action and walks back.

Measured, from `Aq40Multipliers.cpp`:

> Measured at 1441 `A:follow` against 1444 `A:aq40 cthun entry` for one bot: alternating one for
> one, each yanking the destination to its own idea of where the bot belongs.
>
> Returning `true` from the entry action does not stop it; follow runs in a separate pass. The
> only thing that does is zeroing it here.

And on why it matters:

> …with a beam that chains 10y and multiplies 1.5x a jump, a formation that will not hold still
> is the same as no formation.

**Fix.** Zero generic movement in a multiplier while the formation must hold. Not `return true`.
See [01-movement-and-positioning.md](01-movement-and-positioning.md#m1-stop-bots-following-during-a-positioning-phase).

**And scope the suppression to the phase.** The same comment warns:

> Only during the beam phase. In phase 2 melee have to reach the body, and killing generic
> movement there would leave them standing in the ring swinging at air.

---

## G2. Your suppression multiplier does not apply before the pull

**Symptom.** Everything works once the boss is engaged. During the approach and the pre-pull
staging, the bots ignore you completely.

**Cause.** Your multiplier's early-out is `if (!HasActiveNamedUnit(botAI, {"c'thun", …})) return 1.0f;`
— and from the staging position the boss is 90+ yards away and not detectable. The gate never
opens, so nothing is suppressed and follow runs unopposed.

**Fix.** Put a geometric gate *above* the encounter-unit gate, one that does not need to see
anything:

```cpp
if (action && IsInAq40(bot) && !bot->IsInCombat() && !IsAq40Action(action->getName()) &&
    IsGenericMovementAction(action))
{
    float const dx = bot->GetPositionX() + 8578.80f;
    float const dy = bot->GetPositionY() - 1986.20f;
    // Up on the chamber rim and in the approach, which is where staging happens.
    // The height test keeps the raid free to path normally on the ramp below.
    if (dx*dx + dy*dy < 130.0f*130.0f && std::fabs(bot->GetPositionZ() - 108.9f) < 12.0f)
        return 0.0f;
}
```

---

## G3. Your `MoveTo`/`MoveNear` silently did nothing

**Symptom.** The log says `moved=0` while the bot visibly travels across the room.

**Cause.** `IsWaitingForLastMove(priority)` — a move already in flight at equal or higher
`MovementPriority` makes your call a no-op returning `false`. Meanwhile the *old* move is still
executing, so the bot keeps travelling to somewhere you did not choose.

The worked case, from `Aq40Multipliers.cpp` (Twins tank stations):

> The station action could not correct it either: the bot was still executing that path, so its
> own `MoveNear` was refused by `IsWaitingForLastMove` and logged `moved=0` while the tank
> sailed across the room. Hence `station_wait` reading `veklor_distance` 1, 7, 22, 32, 21, 11, 4
> with `moved=0` on every single line.

**Fix.** Stop whatever issued the competing move (usually a multiplier), rather than raising
your own priority in an arms race. A distance readout that changes while `moved=0` is the
signature — look for who else moved the bot, not for a bug in your own action.

---

## G4. `avoid aoe` will walk your tank out of the boss

**Symptom.** The tank whose job is to stand in the Blizzard keeps leaving it.

**Cause.** `avoid aoe` fires at relevance ~90, above your `ACTION_RAID` station action. It does
not know that standing in the fire is the job.

From `Aq40Multipliers.cpp`:

> Including "avoid aoe", which was deliberately let through and is the reason the tanks would
> not stay on Vek'lor. Vek'lor's Blizzard is centred on Vek'lor. The tank whose entire job is to
> stand on him is therefore inside it permanently…
>
> Standing in the Blizzard is the job. That is what the healers are for.

**Fix.** For roles that must eat the AoE, zero *everything* including `avoid aoe`:

```cpp
if (isTankPairMember && (IsGenericPressureAction(action) || IsGenericMovementAction(action)))
    return 0.0f;
```

For roles that must still dodge other things, exempt by name:

```cpp
if (isAssignedHealer && IsGenericMovementAction(action) &&
    !IsActionNamed(action, {"aq40 twin healer anchor", "aq40 twin avoid hazard",
                            "aq40 twin avoid veklor", "avoid aoe"}))
    return 0.0f;
```

---

## G5. "The boss has a target" does not mean the encounter started

**Symptom.** The raid sprints across the room and pulls a boss nobody engaged, while still
fighting trash. Every per-boss trigger reads as active.

**Cause.** `unit->GetTarget()` is set on a boss idling in its own room.

From `Aq40BossHelper.h`:

> …so every per-boss "\<boss\> active" trigger read as live while the raid was still on the
> trash — and those triggers drive `AttackAction`s, so the raid pulled the boss on sight. It
> also suppressed the trash logic, since `IsTrashEncounterActive` stands down whenever a boss
> encounter looks active.

**Fix.** `IsEngagedWithRaid()` — require `IsInCombat()` **and** a victim that resolves to a
player (or a player's pet) in your instance. Keep the two sets separate: `GetEncounterUnits`
for awareness, `GetActiveCombatUnits` for engagement. See
[03-targeting-and-threat.md](03-targeting-and-threat.md#t7-deciding-the-encounter-has-actually-started).

---

## G6. `"find target"` is name-based

`AI_VALUE2(Unit*, "find target", "supremus")` matches on **lowercased creature name**, not
entry. Consequences:

- Typos and capitalisation slips fail silently and return `nullptr` forever.
- Bosses that share a name with an add, or that rename between phases, resolve wrongly.
- Localised servers will not match at all.

Entry-based alternatives when it matters:
`GetFirstAliveUnitByEntry(botAI, entry)` (`RaidBossHelpers.cpp`),
`Aq40BossHelper::Twin::FindUnitByEntry(botAI, units, entry)`.

The same applies to `botAI->HasAura("chill", unit)` — always try the spell-id form first and
keep the name as a fallback. See
[04-hazards-and-avoidance.md](04-hazards-and-avoidance.md#spell-id-lookup-with-a-name-fallback).

---

## G7. A bot ended up **inside** the boss

**Cause.** `YoggSaronBossRoomMovementCheatAction` teleports the bot to the skull target's exact
coordinates:

```cpp
return bot->TeleportTo(bot->GetMapId(), currentSkullUnit->GetPositionX(),
                       currentSkullUnit->GetPositionY(), currentSkullUnit->GetPositionZ(),
                       bot->GetOrientation());
```

No offset, no hitbox allowance. The bot lands at the model origin — inside the boss.

**Related.** `GetNearestPlayerInRadius`'s doc comment flags the same class of error:

> Distance is measured by `GetExactDist2d()`, which does not take into account player hitboxes
> (1.5y).

**Fix.** Offset the destination by at least the combined hitboxes along the bearing from the
target, the way `GetRadialMovePosition(bot, source, desiredDistance)`
(`Aq40Helpers_Shared.cpp`) does — it also handles the degenerate case where the bot is already
exactly on the source and there is no bearing to use:

```cpp
if (len < 0.1f) { dx = std::cos(bot->GetOrientation()); dy = std::sin(bot->GetOrientation()); len = 1.0f; }
```

Do not teleport a bot to a unit's raw position.

---

## G8. Resistance is auras and strategies, not gear

There is no gear-swapping resistance code. `<boss> fire resistance` actions are all
`BossFireResistanceAction(ai, "<boss name>")` from
[BossAuraActions.h](../../src/Ai/Base/Actions/BossAuraActions.h) — one line in an `ActionContext`.

The heavier AQ40 version (`aq40 manage resistance strategies`) toggles **class strategies**:

```cpp
botAI->ChangeStrategy("+rnature", BotState::BOT_STATE_COMBAT);        // hunter
botAI->ChangeStrategy("+nature resistance", BotState::BOT_STATE_COMBAT);  // shaman totem
```

**The trap:** you must track what *you* enabled and only disable that.

```cpp
managedState.natureCombatEnabled = true;      // remember it was us
// … later …
else if (managedState.natureCombatEnabled || managedState.natureNonCombatEnabled) { /* remove */ }
```

Without the flag you will strip a strategy the player set deliberately. And it must be toggled
in **both** `BOT_STATE_COMBAT` and `BOT_STATE_NON_COMBAT` — half of it is a bot that buffs and
then drops the buff on pull.

---

## G9. A stale raid marker re-pulls the boss

**Symptom.** After a wipe, the raid walks back up the ramp and pulls the boss on sight.

**Cause.** From `Aq40Helpers_Shared.cpp`:

> A raid icon is not just decoration to a bot raid: `AttackRtiTargetAction` sends every
> non-healer at whatever wears the skull, and it does not require combat to do so. The Skeram
> code sets a skull on the boss but never takes it off (only C'Thun and the Twins clear theirs),
> so after an attempt the marker sits on him permanently…

**Fix.** `ClearStaleEncounterMarkers`. Note the care it takes: it only clears icons that are on
a known encounter unit which is *not* currently engaged with us, so a marker a human placed on
something else survives.

**If you set an icon, write the clear at the same time.**

---

## G10. Your slot assignment rotates the entire raid

**Symptom.** Bots swap positions constantly during a spread.

**Cause.** The roster you index into changes size. `slotIndex` is a position in a sorted vector;
if one bot drops out of the filter, everyone after it shifts by one and the whole ring rotates.

**Fix.** Filter only on things that change rarely — role, class, alive. Never on in-combat,
in-range, has-aura, has-line-of-sight. Sort by GUID so the order itself never changes:

```cpp
std::sort(v.begin(), v.end(), [](Player* a, Player* b) { return a->GetGUID() < b->GetGUID(); });
```

And handle an unsafe slot by **nudging your own angle**, not by re-picking a slot —
`ICCActions_BPC.cpp` steps π/18 at a time, up to 18 times.

---

## G11. Do not cap a roster at a fixed size

From `Aq40BossHelper.h`:

> Every healer in the instance group, not a fixed four. A raid brings as many as it likes — 12
> in Mike's — and capping the roster at four left the other eight with no side, no anchor and no
> duty, healing whoever happened to be near them while a tank died.

Derive from `size()`. `SideSplit() = (size() + 1) / 2`.

---

## G12. Returning `true` from a bookkeeping action stops the bot fighting

Actions that only maintain state (`manage phase timer`, `erase timers and trackers`) sit at
`ACTION_EMERGENCY + 10/11` so they run first — and therefore **must** return `false`, or they
consume the tick every frame and the bot never attacks.

```cpp
supremusPhaseTimer.try_emplace(instanceId, std::time(nullptr));
return false;                            // <- not true
```

The `erase` variant returns `true` only if it actually erased something, so an idle raid is not
charged a tick.

---

## G13. Some strategies in the tree are switched off

Do not assume something works because the file exists. Currently disabled in
`Naxx/NaxxStrategy.cpp`:

- **Heigan the Unclean** — the whole dance. `heigan dance melee` / `heigan dance ranged` are
  registered actions, but `NaxxActions_Heigan.cpp` is **100% commented out**, as are the two
  `TriggerNode`s and `HeiganDanceMultiplier`.
- **Patchwerk** — `patchwerk tank` / `patchwerk ranged` / `patchwerk non-tank` triggers commented out.
- **Gothik** — `GothikGenericMultiplier` commented out.

Also `Aq40Helpers::ShouldSuppressTwinPrePullMaintenance` is a stub that unconditionally returns
`false` with `(void)` casts on all three parameters.

If you are asked to "fix Heigan", the first question is whether to revive the commented code or
start fresh — it was disabled for a reason that is not recorded, which is itself a lesson: **when
you disable something, write down why.**

---

## G14. `getMSTime()` wraps

It is a `uint32` millisecond counter. `now - then` is wrong across the wrap.

```cpp
getMSTimeDiff(stampedAtMs, nowMs)         // correct
nowMs - stampedAtMs                        // wrong
```

And treat `0` as "never stamped", because it is indistinguishable from a real timestamp
otherwise:

```cpp
return stampedAtMs && getMSTimeDiff(stampedAtMs, nowMs) <= windowMs;
```

---

## G15. `GetCurrentSpell(CURRENT_GENERIC_SPELL)` misses channels

Channelled spells live in `CURRENT_CHANNELED_SPELL`. Check both:

```cpp
Spell* spell = unit->GetCurrentSpell(CURRENT_GENERIC_SPELL);
if (!spell) spell = unit->GetCurrentSpell(CURRENT_CHANNELED_SPELL);
```

And because the cast window is short, prefer cast-**or**-aura-**or**-name, as
`IsSarturaSpinning` does.

---

## G16. One mechanic has four spell ids

10N / 25N / 10H / 25H, plus occasional extras (`SPELL_MALLEABLE_GOO_BALCONY`). Every id test
must cover the whole set. The house helpers take initialiser lists precisely for this:

```cpp
NaxxSpellIds::MatchesAnySpellId(info, {Decimate10, Decimate25, Decimate25Alt})
Aq40SpellIds::HasAnyAura(botAI, unit, {TwinMutateBug, TwinVirulentPoisonProc})
```

---

## G17. `ImbueItem` does not reliably move the item

Use it for the animation; move the item yourself in a deferred event. See
[07-teleports-vehicles-portals-items.md](07-teleports-vehicles-portals-items.md#-imbueitem-does-not-reliably-move-the-item).

---

## G18. Removing follow without owning the restore

`botAI->ChangeStrategy("-follow", …)` persists. If the bot dies, zones, or the encounter resets
before your action runs again, follow stays off for the rest of the session.

From `Aq40Actions_Shared.cpp`:

> The staging action takes the follow strategy off the bot so it can position it, [and] …
> mid-approach keeps follow removed for the rest of the session. This is the … doubles as the
> un-stage — leave the zone and the raid goes back to following.

Prefer a multiplier (scoped to the tick) over strategy removal (scoped to the session). If you
must remove it, add an unconditional restore path that does not depend on your action running.

---

## G19. Engine quirks that already have named workarounds

Do not re-diagnose these:

- **Supremus' hitbox is bugged** for Killing Spree. `SupremusHitboxIsBuggedMultiplier` zeroes
  `CastKillingSpreeAction` for rogues while Supremus is up.
- **Blink teleports along the caster's facing**, so a bot must face away from the destination
  before blinking backwards — `BWLActions.cpp` ~895.
- **Bots can enter a vehicle while mounted** and end up broken — cancel the mount aura after
  boarding.
- **Bots do not take fall damage**, so a "jump down" strategy is free for them and is sometimes
  implemented only to mimic the human strat (`MotherShahrazPositionRangedUnderPillarAction`
  says so in a comment).
- **Obsidian Eradicator/Nullifier drain mana map-wide**, so distance does not protect you — the
  counter is starvation, not positioning (`Aq40BossHelper.h`, `FeedsObsidianDrainer`).
