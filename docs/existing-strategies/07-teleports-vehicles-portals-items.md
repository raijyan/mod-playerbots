# 07 — Teleports, vehicles, portals and items

Everything in this chapter is where the bot stops being a pathing problem and starts being a
client-protocol problem.

---

## P1. Teleport a bot to a fixed recovery position

The `fall from floor` family — five of them (`kologarn`, `auriaya`, `thorim`, `yogg-saron`,
`emalon`). All identical:

```cpp
bool KologarnFallFromFloorAction::Execute(Event)
{
    return bot->TeleportTo(bot->GetMapId(),
        ULDUAR_KOLOGARN_RESTORE_POSITION.GetPositionX(),
        ULDUAR_KOLOGARN_RESTORE_POSITION.GetPositionY(),
        ULDUAR_KOLOGARN_RESTORE_POSITION.GetPositionZ(),
        ULDUAR_KOLOGARN_RESTORE_POSITION.GetOrientation());
}
```

The work is all in the **trigger** — deciding "this bot has fallen and cannot path back". The
usual test is a Z threshold plus a proximity/room test; check the matching
`<Boss>FallFromFloorTrigger` in `UldTriggers.cpp` before writing a new one.

`Kologarn eyebeam action` (`UldActions.cpp` ~1340) also uses `TeleportTo` for lateral
repositioning, which is the "no path exists in time" escape hatch rather than a fall recovery.

### Teleport hygiene

- `bot->IsBeingTeleported()` must be checked before doing anything positional — it is in
  `IsAq40FollowRecoveryCandidate`'s guard list alongside `HasUnitState(UNIT_STATE_IN_FLIGHT)`
  and `IsNonMeleeSpellCast(true)`.
- A teleport does not clear the MotionMaster. If the bot was mid-`MoveTo`, it will try to walk
  back. `TryRecoverAq40FollowState` clears `MotionMaster`, `UNIT_STATE_CHASE` and
  `UNIT_STATE_FOLLOW` — do the same after any forced relocation.
- `JumpTo(...)` is the non-teleport alternative where a drop is intended, used by
  `ThaddiusMoveToPlatformAction` to fall between platform levels. The commented-out `TeleportTo`
  immediately beside it records that jumping was chosen over teleporting deliberately.

---

## P2. Vehicles

```cpp
bool FlameLeviathanEnterVehicleAction::EnterVehicle(Unit* vehicleBase, bool moveIfFar)
{
    float dist = bot->GetDistance(vehicleBase);
    if (dist > INTERACTION_DISTANCE && !moveIfFar) return false;
    if (dist > INTERACTION_DISTANCE)              return MoveTo(vehicleBase);

    botAI->RemoveShapeshift();
    // Use HandleSpellClick instead of Unit::EnterVehicle to handle special vehicle script (ulduar)
    vehicleBase->HandleSpellClick(bot);

    if (!bot->IsOnVehicle(vehicleBase)) return false;

    // dismount because bots can enter vehicle on mount
    WorldPacket emptyPacket;
    bot->GetSession()->HandleCancelMountAuraOpcode(emptyPacket);
    return true;
}
```

Four non-obvious requirements, all of which cost someone an evening:

1. **`HandleSpellClick`, not `Unit::EnterVehicle`** — the latter bypasses the vehicle's script.
2. **`RemoveShapeshift()` first** — a druid in form cannot board.
3. **Verify with `IsOnVehicle`** — the click can silently fail.
4. **Cancel the mount aura afterwards** — a bot can board while mounted and end up in a broken
   state.

Selection: `AI_VALUE(GuidVector, "nearest vehicles far")`, skipping
`UNIT_FLAG_NOT_SELECTABLE`, nearest-first. Triggers: `FlameLeviathanOnVehicleTrigger`
(`bot->GetVehicle()` / `GetVehicleBase()` against a whitelist of vehicle entries),
`FlameLeviathanVehicleNearTrigger`.

Other vehicle-shaped encounters: `eoe fly drake` / `eoe drake attack` (EoE),
`icc cannon fire` / `icc gunship enter cannon` (ICC).

---

## P3. Portals

Two mechanisms, depending on whether the portal is a creature or a gameobject:

```cpp
Creature* assignedPortal = bot->FindNearestCreature(NPC_DESCEND_INTO_MADNESS, 2.0f, true);
return assignedPortal->HandleSpellClick(bot);            // Yogg-Saron
```
```cpp
portal->Use(bot);                                        // RS Halion twilight portal (GameObject)
```

`YoggSaronUsePortalAction` **re-adds `FollowMasterStrategy`** on the way through, because
`YoggSaronMoveToEnterPortalAction` removed it to do the approach. Whoever removes follow owns
giving it back — see
[01-movement-and-positioning.md](01-movement-and-positioning.md#m2-take-the-follow-strategy-off-a-bot-entirely-and-give-it-back).

Actions: `enter twilight portal` / `exit twilight portal` (RS), `rs halion enter portal`,
`yogg-saron move to enter portal action` / `move to exit portal action` / `use portal action`,
`icc valithria portal`.

**Bots on the far side of a portal cannot see the boss.** RS solves this by stamping boss health
server-side; see [06-server-side-hooks-and-cheats.md](06-server-side-hooks-and-cheats.md#k3-poll-a-creature-every-tick).

---

## P4. Using items

**On a unit** — the animation-bearing path:

```cpp
Item* targetSpine = bot->GetItemByEntry(ITEM_NAJENTUS_SPINE);
Unit* targetNajentus = botAI->GetUnit(najentusGuid);
if (targetSpine && targetNajentus)
    botAI->ImbueItem(targetSpine, targetNajentus);
```
(`HighWarlordNajentusThrowImpalingSpineAction`, wrapped in `AddTimedEvent` with `urand(500,1500)`
jitter.)

**On a gameobject** — `ImbueItem` cannot target a GO, so build the packet by hand:

```cpp
WorldPacket packet(CMSG_USE_ITEM);
packet << bagIndex << slot << cast_count << spellId << item_guid << glyphIndex << castFlags;
packet << (uint32)TARGET_FLAG_GAMEOBJECT;
packet << generator->GetGUID().WriteAsPacked();
bot->GetSession()->HandleUseItemOpcode(packet);
```
(`LadyVashjPassTheTaintedCoreAction::UseCoreOnNearestGenerator`. `spellId` is pulled from the
first non-zero `core->GetTemplate()->Spells[i].SpellId`; guarded by `bot->CanUseItem(core)` and
`!bot->IsNonMeleeSpellCast(false)`.)

**Clickable gameobject:** `go->Use(bot)` — `GO_NAJENTUS_SPINE`, `GO_SHADOW_TRAP` (BT),
Manticron Cube (`MagActions.cpp`), obedience crystal (Naxx).

**Item lookups:** `bot->GetItemByEntry(entry)`, `botAI->HasItemInInventory(entry)`,
`bot->HasItemCount(entry, count, false)`, `bot->CanUseItem(item) == EQUIP_ERR_OK`.

### ⚠️ ImbueItem does not reliably move the item

Documented in `SSCActions.cpp`:

> `ImbueItem()` is inconsistent in causing the receiver bot to receive the core and the giver
> bot to remove the core, so `ScheduleTransferCoreAfterImbue()` creates the core on the receiver
> and removes it from the giver, with `ImbueItem()` called primarily for the throwing animation.

The workaround, and the shape to copy for any bot-to-bot item hand-off:

```cpp
botAI->AddTimedEvent([receiverGuid, giverGuid]()
{
    Player* receiverPlayer = ObjectAccessor::FindPlayer(receiverGuid);
    Player* giverPlayer    = ObjectAccessor::FindPlayer(giverGuid);
    if (!receiverPlayer) return;

    if (!receiverPlayer->HasItemCount(ITEM_TAINTED_CORE, 1, false))
    {
        ItemPosCountVec dest;
        if (receiverPlayer->CanStoreNewItem(NULL_BAG, NULL_SLOT, dest, ITEM_TAINTED_CORE, 1) == EQUIP_ERR_OK)
            receiverPlayer->StoreNewItem(dest, ITEM_TAINTED_CORE, true,
                Item::GenerateItemRandomPropertyId(ITEM_TAINTED_CORE));
    }

    if (giverPlayer)
        if (Item* item = giverPlayer->GetItemByEntry(ITEM_TAINTED_CORE))
            if (giverPlayer->HasItemCount(ITEM_TAINTED_CORE, 1, false))
                giverPlayer->DestroyItem(item->GetBagSlot(), item->GetSlot(), true);
}, /*delayMs*/ 1500);
```

Note it captures **GUIDs and re-resolves**, checks the receiver does not already hold one
(so a double-fire cannot duplicate the item), and destroys from the giver only after confirming
the count.

---

## P5. Equipment swaps

`kael'thas sunstrider reequip gear` / `use legendary weapons` / `loot legendary weapons`
(`TKActions.cpp` ~1727–1765) — swaps to the dropped legendary weapons and back, gated on
`bot->CanUseItem(mainHand/offHand) == EQUIP_ERR_OK` and on not already having
`SPELL_ARCANE_BARRIER`.

Resistance *gear* is not swapped anywhere; resistance is handled entirely as auras/strategies —
see [08-gotchas.md](08-gotchas.md#g8-resistance-is-auras-and-strategies-not-gear).
