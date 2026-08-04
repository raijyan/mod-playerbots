# Existing bot raid strategies — a guidebook keyed by mechanic

**Purpose.** Before you write anything for a boss, look up the *mechanic* here, not the boss.
Almost every problem you will hit ("stop the bots following while I position them", "hold a
formation without owning the tick", "one bot should own the raid markers", "a bot ended up
inside the boss") has already been solved somewhere in `src/Ai/Raid/`. This directory tells
you where, what the canonical shape is, and which variants were tried and abandoned.

The cost this exists to avoid: rediscovering a solved problem, and inventing a novel
mechanism that a reviewer will (correctly) ask you to replace with the existing one.

## How to use it

1. Find your mechanic in the lookup table below.
2. Open the linked chapter. Each entry gives: the canonical implementation, the shape to copy,
   the variants in the tree, and the traps.
3. Copy the canonical shape. Do not invent a new one without a reason you can state in the PR.
4. Before you push, read [10-antipatterns-and-pr-risks.md](10-antipatterns-and-pr-risks.md).

## Scope of the audit

All 20 raid strategies under [src/Ai/Raid/](../../src/Ai/Raid/) (~92k lines, 285 files),
plus the shared layer and the base action library they draw on. Dungeon strategies
([src/Ai/Dungeon/](../../src/Ai/Dungeon/), ~9k lines) follow the identical architecture and are
covered by the same chapters.

> **Note on AQ40.** The audit was carried out against a working branch that includes the
> in-progress `src/Ai/Raid/Aq40/` strategy, which is **not yet on `master`**. AQ40 is quoted
> heavily throughout — it carries the most thoroughly documented reasoning in the codebase,
> particularly on movement suppression, formation holding and encounter-start detection — so
> those passages are forward references until that work lands. Every other instance referenced
> here (BT, ICC, SSC, TK, Kara, Naxx, RS, Uld, BWL, ZA, Hyjal, Gruul, Mag, MC, VoA, EoE, OS,
> Ony, Aq20) is present on `master`, and every claim about them was verified against it.
>
> The two links that currently dangle are `src/Ai/Raid/Aq40/Aq40Multipliers.cpp` and
> `src/Ai/Raid/Aq40/Util/Aq40Helpers_Shared.cpp`. The lessons they encode are transferable and
> stand on their own — they are the reason those sections exist.

## Mechanic lookup

### Movement, positioning, formation → [01-movement-and-positioning.md](01-movement-and-positioning.md)

| I want to… | Canonical implementation |
|---|---|
| Stop bots following during a positioning phase | `IsGenericMovementAction()` zeroed in a multiplier — `Aq40Multipliers.cpp` |
| Hold a formation without the tick being stolen back | `Aq40CthunMultiplier::GetValue` "while the Eye is up" block |
| Take the follow strategy off a bot entirely, and give it back | `Aq40Helpers::TryRecoverAq40FollowState`, `YoggSaronBossRoomMovementCheatAction` |
| Anchor the boss at a fixed spot (tank job) | `HighWarlordNajentusTanksPositionBossAction::Execute` |
| Walk a bot through a multi-leg route | `TankPositionState` + `shahrazTankStep` (BT), `tidewalkerTankStep` (SSC) |
| Spread the raid on a ring, one slot each | `IccBpcEmpoweredVortexAction` (GUID-sorted slot index) |
| Spread by distance rather than by slot | `HighWarlordNajentusDisperseRangedAction` + `FleePosition` |
| Find a safe spot away from hazards, with a safe path | `KarazhanHelpers::TryFindSafePositionWithSafePath`, `SupremusMoveAwayFromVolcanosAction` |
| Orbit / rotate around the boss | `Aq40CthunAvoidDarkGlareAction`, `RotateAroundTheCenterPointAction` |
| Kite a fixating boss | `SupremusKiteBossAction` |
| Stage the raid before a pull | `Aq40BossHelper::Twin::ShouldStageForPull` + `GetStagingPoint` |
| Stand behind the boss / avoid frontal | `RearFlankAction` (base library) |

### Assignment, ownership, markers → [02-coordination-and-assignment.md](02-coordination-and-assignment.md)

| I want to… | Canonical implementation |
|---|---|
| Have exactly one bot do a raid-wide job | `IsMechanicTrackerBot(bot, mapId)` — `RaidBossHelpers.cpp` |
| Set a raid target icon | `MarkTargetWithSkull()` etc. — `RaidBossHelpers.cpp` |
| Set an icon *and* clear it afterwards | `Aq40Helpers::SetRaidTargetIcon` / `ClearStaleEncounterMarkers` |
| Find the main tank / an assist tank | `GetGroupMainTank`, `GetGroupAssistTank` — `RaidBossHelpers.cpp` |
| Build a richer tank roster (primary + backups) | `Aq40BossHelper::GetEncounterPrimaryTank` / `GetEncounterBackupTank` |
| Assign N bots to N jobs, stably | GUID sort → index. `Aq40BossHelper::Twin::GetTankPairAssignments` |
| Split healers between two sides | `Aq40BossHelper::Twin::HealerAssignments` |
| Chain hunter misdirects to different tanks | `SupremusMisdirectBossToMainTankAction::Execute` |
| Rotate the raid through debuff groups | `GetGurtoggRangedRotationGroups` / `GetGurtoggActiveRotationGroup` |

### Targeting, threat, kill order → [03-targeting-and-threat.md](03-targeting-and-threat.md)

| I want to… | Canonical implementation |
|---|---|
| Override which target the bots attack | `<boss> choose target` actions, e.g. `Aq40CthunChooseTargetAction` |
| Enforce a kill order | `FourHorsemenAttackInOrderAction::Execute` |
| Stop the generic targeting engine from fighting me | zero `"aq40 choose target"` / `DpsAssistAction` / `TankAssistAction` in a multiplier |
| Stop DPS pulling before the tank has aggro | `Aq40BossHelper::ShouldWaitForEncounterTankAggro` |
| Hold DPS for N seconds after a phase change | `<boss>DpsWaitTimer` map + `WaitForDpsMultiplier` |
| Hold burst cooldowns until the boss is engaged | `IsDpsCooldownAction()` + `DelayDpsCooldownsMultiplier` |
| Detect "the encounter has actually started" | `Aq40BossHelper::IsEngagedWithRaid` (**not** `unit->GetTarget()`) |

### Hazards and avoidance → [04-hazards-and-avoidance.md](04-hazards-and-avoidance.md)

| I want to… | Canonical implementation |
|---|---|
| Avoid a dynamic-object AoE (Blizzard, Consecration…) | `GetDynamicObjectPositions()` — `RaidBossHelpers.cpp` |
| Avoid a hazard that is a spawned creature | `GetCreatureListWithEntryInGrid` + radius test — `SupremusMoveAwayFromVolcanosAction` |
| Avoid a cone / beam pointed at me | `Aq40CthunAvoidDarkGlareAction` (cross-product side test) |
| React to a debuff on myself | `ThaddiusMovePolarityAction`, `MoveAwayFromPlayerWithDebuffAction` |
| React to a debuff on someone else | `HighWarlordNajentusPlayerIsImpaledTrigger` |
| Know where an AoE *will* land, before it does | server-side `OnSpellCast` hook → instance state. `ICCScripts.cpp` |

### Phases, timers, shared state → [05-phases-timers-and-state.md](05-phases-timers-and-state.md)

| I want to… | Canonical implementation |
|---|---|
| Know what phase the boss is in | flags/auras first (`ThaddiusBossHelper::IsPhasePet`); boss-AI cast only as last resort |
| Read the boss's own `EventMap` | `GenericBossHelper<BossAiType>` — `NaxxBossHelper.h` (**fragile, see ch. 10**) |
| Keep per-instance state | `std::unordered_map<uint32 /*instanceId*/, T>` in `<X>Helpers.cpp` or `<X>Scripts.cpp` |
| Keep per-bot state | `std::unordered_map<ObjectGuid, T>` |
| Time something from an event | `<x>Timer.try_emplace(instanceId, std::time(nullptr))` via a `manage … timer` action |
| Clean state up after a wipe/kill | `<instance> erase timers and trackers` action, fired by a not-in-combat trigger |
| Do something after a delay | `botAI->AddTimedEvent(lambda, delayMs)` |

### Server-side hooks and cheats → [06-server-side-hooks-and-cheats.md](06-server-side-hooks-and-cheats.md)

| I want to… | Canonical implementation |
|---|---|
| Observe a spell the bots cannot see coming | `AllSpellScript::OnSpellCast` / `OnSpellPrepare` — `ICCScripts.cpp` |
| Poll a boss every tick server-side | `AllCreatureScript::OnAllCreatureUpdate` — `ICCScripts.cpp`, `RSScripts.cpp` |
| Free instance state when the map dies | `AllMapScript::OnDestroyMap` |
| Change the encounter rather than the bots | `RSScripts.cpp` (root/teleport/god-mode Halion) — **last resort** |
| Have bots skip a mechanic outright | `<x> cheat action` + `botAI->HasCheat(BotCheatMask::raid)` |

### Teleports, vehicles, portals, items → [07-teleports-vehicles-portals-items.md](07-teleports-vehicles-portals-items.md)

| I want to… | Canonical implementation |
|---|---|
| Recover a bot that fell off the map | `KologarnFallFromFloorAction` (`TeleportTo` a fixed restore position) |
| Put bots in a vehicle | `FlameLeviathanEnterVehicleAction` |
| Send bots through a portal | `YoggSaronUsePortalAction` (`HandleSpellClick`), `enter twilight portal` (RS) |
| Use a quest/encounter item | `botAI->ImbueItem(item, target)` — `HighWarlordNajentusThrowImpalingSpineAction` |
| Use a clickable gameobject | `GameObject::Use(bot)` — same file |
| Pass an item between bots | `LadyVashjPassTheTaintedCoreAction` + `GetCoreHandlers` |

### The traps → [08-gotchas.md](08-gotchas.md)

Read this one even if you think you know what you are doing. It is the list of things that
have already cost someone an evening.

### Reference

- [00-architecture.md](00-architecture.md) — the mandatory file/class shape and the wiring checklist.
- [09-index-by-instance.md](09-index-by-instance.md) — what already exists, per instance and per boss.
- [10-antipatterns-and-pr-risks.md](10-antipatterns-and-pr-risks.md) — what a reviewer will object to, and what is already in the tree that you should not copy.

## Maintaining this

When you solve something new, add the mechanic to the right chapter and to the table above.
When you discover a trap, add it to `08-gotchas.md` **while you are still in the debugger** —
that is when it is cheap. Delete entries that stop being true rather than annotating them as
outdated.
