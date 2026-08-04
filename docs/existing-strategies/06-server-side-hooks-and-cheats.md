# 06 — Server-side hooks, and cheats

Some things a bot cannot know from its own perspective: where an instant AoE is about to land,
what a boss's internal counter is, whether an event fired at all. For those, the module
installs `ScriptMgr` hooks that observe the server and stamp shared instance state, which the
bot AI then reads normally.

Below that there is a second, more invasive tier: changing the **encounter** rather than the
bots. That tier is a last resort and should be argued for explicitly in a PR.

---

## K1. Where the hooks live and how they register

```
<X>Scripts.h/.cpp                     e.g. ICC/ICCScripts.cpp, RS/RSScripts.cpp,
                                           Aq40/Aq40Scripts.cpp, Mag/Util/MagScripts.cpp,
                                           TK/Util/TKScripts.cpp, Hyjal/Util/HyjalScripts.cpp
```

Each exposes `void AddSC_<Name>BotScripts();`, declared and called in
[src/Script/Playerbots.cpp](../../src/Script/Playerbots.cpp) (~lines 516–541). A new hook file
needs entries in **both** places or it silently never runs.

```cpp
void AddSC_IcecrownBotScripts()
{
    new IccPutricideListenerScript();
    new IccRotfaceListenerScript();
    new IccLichKingListenerScript();
    new IccBossStateResetScript();
    new IccMapCleanupScript();
}
```

Only six of the twenty raid strategies have a `Scripts` file. **Most encounters do not need
one** — reach for it only when the information genuinely is not observable from the bot.

---

## K2. Observe a spell the bots cannot react to

```cpp
class IccPutricideListenerScript : public AllSpellScript
{
    void OnSpellCast(Spell* spell, Unit* caster, SpellInfo const* spellInfo, bool) override
    {
        if (spellInfo->Id != SPELL_MALLEABLE_GOO_10N && /* … all difficulty variants … */)
            return;

        // Malleable Goo is cast triggered, so m_UniqueTargetInfo is not yet
        // populated at this point; read the explicit unit target directly.
        Unit* target = spell->m_targets.GetUnitTarget();
        if (!target || !target->IsPlayer())
            return;

        auto& impacts = IcecrownHelpers::IccState(caster->GetMap()->GetInstanceId()).malleableGoo;
        impacts.push_back({ target->GetPosition(), getMSTime() });

        // Evict stale entries to keep the list bounded. Retention covers the
        // longest consumer window (Festergut avoid: 8s) + slack.
        impacts.erase(std::remove_if(impacts.begin(), impacts.end(),
            [now](auto const& i) { return getMSTimeDiff(i.castTime, now) > 9000; }), impacts.end());
    }
};
```

Four things to copy:

1. **Filter on spell id first**, and cover every difficulty variant (10N/25N/10H/25H, plus
   oddities like `SPELL_MALLEABLE_GOO_BALCONY`). This hook fires for *every spell cast on the
   server*; the id test must be the first thing.
2. **`m_targets.GetUnitTarget()` for triggered spells** — `m_UniqueTargetInfo` is not populated
   yet at `OnSpellCast`.
3. **Bound the container**, with the retention window derived from the longest consumer.
4. **`OnSpellPrepare` vs `OnSpellCast`**: `IccLichKingListenerScript` documents it —
   *"OnSpellPrepare fires at cast START (Spell::prepare). OnSpellCast fires…"* later. Use
   `OnSpellPrepare` when the bots need the cast time to react in.

Consumers read a filtered, time-windowed view:

```cpp
std::vector<Position> ActiveGooPositions(uint32 instanceId, uint32 lifetimeMs);
```

Other users: `Aq40Scripts.cpp` stamps Twin teleport / Blizzard / Arcane Burst / Heal Brother /
Explode Bug, plus the exploding bug's **source position**, which the bots would otherwise lose
when the bug dies.

### Picking the right spell to listen for

The AQ40 comment is the model for how much rigour this deserves:

> 800, not 799, and only from an emperor. 799 is the *request*: a SCRIPT_EFFECT self-cast whose
> handler swaps the pair only if it can resolve the twin, so it can fire without a swap
> happening. 800 is cast triggered on both emperors from inside the committed block in
> `boss_veklor::SpellHit`, after the decision and either side of the two `NearTeleportTo` calls
> — it cannot be reached unless the pair actually traded places.

Read the core script and pick the id that is only reachable once the thing has *actually
happened*. Then dedupe, because "cast on both" means two events per swap.

---

## K3. Poll a creature every tick

```cpp
class IccBossStateResetScript : public AllCreatureScript
{
    void OnAllCreatureUpdate(Creature* creature, uint32) override
    {
        if (!creature || creature->GetMapId() != ICC_MAP_ID || !creature->IsDungeonBoss())
            return;
        // … stamp lastBossCombatMs; reset instance state after a grace period out of combat
    }
};
```

Same rule as K2: **guard on map id first**. This runs for every creature update on the server.

`RSScripts.cpp` uses this hook heavily to mirror otherwise-invisible boss state into
`RsInstanceState` — corporeality index, and boss health stamped every tick *specifically so
that bots on the other side of a portal, who cannot see the boss, can still read it*:

```cpp
// boss health % stamped server-side (visible in every phase),
// read by bots that cannot see the boss through a portal
struct BossHealth { uint8 pct = 0; uint32 stamp = 0; };
```

That is the legitimate, high-value use of this tier: **make invisible state visible**, then let
the bots decide with it.

---

## K4. Free instance state

```cpp
class IccMapCleanupScript : public AllMapScript
{
    void OnDestroyMap(Map* map) override
    {
        if (map->GetId() == ICC_MAP_ID)
            IcecrownHelpers::IccResetInstance(map->GetInstanceId());
    }
};
```

Mandatory for anything holding a per-instance map. Without it the map leaks one entry per
instance created, forever.

---

## K5. Changing the encounter instead of the bots

`RSScripts.cpp` is the far end of the spectrum, and it is worth knowing exactly what it does so
that you know what you are being asked to sign off on when you copy it:

- **Roots the boss and teleports it home** while a twilight tank is alive, so Halion does not
  wander (`SetControlled(true, UNIT_STATE_ROOT)` + `NearTeleportTo(homePosition)`).
- **Grants bots a Magic Barrier aura** while the boss casts a breath, and strips it 1s after —
  `HandleBreathGodMode`, which explicitly excludes tanks and real players
  (`!GET_PLAYERBOT_AI(player) || PlayerbotAI::IsTank(player)` → skip).
- **Dispels and god-modes** through Consumption (`HandleConsumptionDispel`,
  `HandleConsumptionGodMode`).

This makes the encounter beatable by bots that cannot execute the real mechanic. It is a
**capability substitute, not a strategy**, and it comes with three costs a reviewer will name:
the encounter no longer behaves as designed for the humans in the raid; the real mechanic never
gets implemented; and the code is now coupled to core spell ids and creature behaviour.

If you reach for this, say in the PR body why the bot-side solution is not viable.

---

## K6. Cheat actions

The bot-side equivalent: an action that skips a mechanic outright, gated on an opt-in flag.

```cpp
if (!botAI->HasCheat(BotCheatMask::raid))
    return false;
```

`BotCheatMask` ([src/PlayerbotAIConfig.h](../../src/PlayerbotAIConfig.h)):
`none=0, taxi=1, gold=2, health=4, mana=8, power=16, raid=32, food=64`.

Existing cheat actions, all in Ulduar plus one in BT:

| Action | What it does |
|---|---|
| `mimiron cheat action` | `unit->Kill(bot, unit)` on Proximity Mines and Bomb Bots |
| `yogg-saron ominous cloud cheat action` | `target->Kill(bot, target)` on the Ominous Cloud |
| `vezax cheat action` | `bot->SetPower(POWER_MANA, bot->GetMaxPower(POWER_MANA))` |
| `yogg-saron boss room movement cheat action` | `bot->TeleportTo(...)` onto the current skull target |
| `illidan stormrage handle adds cheat` | (BT) |

**Gate the cheat in the trigger, not the action.** That is what Ulduar does — `HasCheat` is the
first line of `MimironCheatTrigger::IsActive`, `VezaxCheatTrigger::IsActive` and
`YoggSaronOminousCloudCheatTrigger::IsActive`, so with cheats off the action is never even
queued. `BotCheatMask::raid` is checked at ~20 sites across `UldTriggers.cpp` / `UldActions.cpp`
and once in `BWLTriggers.cpp`; use it consistently.

⚠️ One ordering wart to avoid copying: `YoggSaronBossRoomMovementCheatTrigger::IsActive` returns
`true` **before** its `HasCheat` check whenever the bot still has `FollowMasterStrategy`, purely
so the action can strip follow. The action then bails at `!HasCheat`. Net effect with cheats
off: follow is removed and this path never restores it. See
[08-gotchas.md](08-gotchas.md#g18-removing-follow-without-owning-the-restore).

⚠️ That action teleports the bot to the skull target's exact coordinates — which is how a bot
ends up standing **inside** the boss model. See
[08-gotchas.md](08-gotchas.md#g7-a-bot-ended-up-inside-the-boss).

---

## K7. Thread safety

Hooks fire on map update threads. State they touch **must** be mutex-guarded:

```cpp
namespace { std::unordered_map<uint32, TwinScriptState> sTwinStateByInstance; std::mutex sStateMutex; }

bool IsTwinTeleportPickupWindow(Player const* bot, uint32 windowMs, uint32 nowMs)
{
    std::lock_guard<std::mutex> guard(sStateMutex);
    TwinScriptState* state = GetState(bot);
    return state && IsRecent(state->lastTeleportAtMs, windowMs, ResolveNow(nowMs));
}
```
(`Aq40Scripts.cpp` — lock held for the whole accessor, result returned by value.)

Contrast `ICCScripts.cpp`'s `IccState(instanceId)`, which locks only to obtain the reference and
then hands it out — callers mutate it unlocked. **Follow the AQ40 shape in new code.**

Plain unguarded globals in `<X>Helpers.cpp` are fine *only* for state written exclusively from
bot AI code. The moment a hook writes it, it needs a mutex.
