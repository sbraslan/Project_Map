# Mining / Pickaxe

**Status:** STATIC COMPLETE / 7 VERIFIED BUGS / EXECUTION LOCKED  
**Phase:** Detection / Mapping Only  
**Source repos:** read-only  
**Writable repo:** Project_Map only

## Scope
- server mining start/cancel/event flow;
- ore vein mapping and ore drops;
- pickaxe practice/mastery/refine;
- mining skill contribution;
- ore refinement;
- death/warp/logout/equipment lifecycle;
- multiplayer ownership/drop behavior;
- quest bindings and deployed mining quest.

## Main roots
- `Project_ServerSRC/game/src/mining.cpp`
- `Project_ServerSRC/game/src/mining.h`
- `Project_ServerSRC/game/src/char.cpp`
- `Project_ServerSRC/game/src/char.h`
- `Project_ServerSRC/game/src/questlua_global.cpp`
- `Project_ServerSRC/game/src/questlua_pc.cpp`
- `Project_Game/share/locale/europe/quest/n_npc/mining.quest`
- `Project_DumpProto/*/item_names.txt`

## Feature gates
- `ENABLE_PICKAXE_RENEWAL` is active.
- `ENABLE_MINING_EVENT` is active.
- Yohara extends ore mapping from 19 to 23 vein types when enabled.

## Normal mining flow
`pc.mining(npc)` -> `CHARACTER::mining(load)`
1. rejects an existing mining event by cancelling it;
2. validates load, map/distance and ore race;
3. requires equipped ITEM_PICK subtype 0;
4. broadcasts DIG_MOTION;
5. creates delayed mining event for 5..15 moves at 2 seconds each.

At event completion:
- player event pointer is cleared via `mining_take()`;
- pick and load are re-resolved;
- ore success chance = 20 + mining skill bonus + pick grade bonus;
- success calls `OreDrop()`;
- `PracticePick()` may increment mastery socket0.

## Pickaxe refine flow
Deployed quest `n_npc/mining.quest` handles pickaxes 29101..29109 and calls global `__refine_pick(item.get_cell())`.

Global binding:
`__refine_pick` -> `_refine_pick` -> `mining::RealRefinePick(ch,item)`.

### BUG-MIN-001 — quest/C++ mastery boundary mismatch makes normal pickaxe refine unreachable
Quest opens the refine path only when:
`item.get_socket(0) == item.get_value(2)`.

But C++ `Pick_Refinable()` returns false when:
`Pick_GetCurExp(item) <= Pick_GetMaxExp(item)`.

Therefore `RealRefinePick()` only accepts mastery strictly greater than the configured max.

`PracticePick()` also behaves around the same boundary:
- at socket0 == max, `Pick_Refinable()` is false;
- a successful practice increments socket0 to max+1;
- only then C++ considers it refinable.

But at max+1, the deployed quest's equality condition no longer matches, so the normal NPC quest no longer enters its refine branch.

Promoted as `BUG-MIN-001`.

## BUG-MIN-002 — mining can resolve after death
Player mining event is cancelled by movement via `OnMove() -> mining_cancel()` and by final character destruction.

However `CHARACTER::Dead()` does not cancel `m_pkMiningEvent`, and the delayed `mining_event` callback does not check `IsDead()`.

If the character dies after mining starts but before the delayed event fires, the callback can still:
- resolve player/load;
- validate pick;
- roll ore success;
- call `OreDrop()`;
- grant pickaxe practice progress.

Promoted as `BUG-MIN-002`.

## Candidates / open work
- `OreRefine()` subtracts 100 raw ore before checking whether the player has enough gold. This is a real local ordering defect, but the current deployed quest tree has not yet shown a reachable `pc.ore_refine/pc.diamond_refine` caller; keep candidate until deployment reachability closes.
- Mining start has an early distance rejection at >1000 and later anti-hack logging only at >2500; the latter branch appears unreachable. Determine whether this is only telemetry dead code or signals a stale intended distance boundary.
- Map/warp lifecycle needs closure: movement cancels mining, but direct warp paths must be checked for explicit cancellation/revalidation.
- Equipment swap/unequip timing needs closure.
- Ore ownership/Battle Field behavior and mining-event integration still need review.
- Pickaxe item proto values/refine progression should be cross-checked when an authoritative proto row is available.

## Runtime
No Mining runtime test may be executed while the global execution lock is active. First future live gate remains `DUNGEON-T09`.


## BUG-MIN-003 — delayed mining uses whichever pickaxe is equipped at completion
The mining event stores only:
- player PID;
- ore-load VID.

It does not store the initiating pickaxe ID/VID.

At event completion it resolves:
`LPITEM pick = ch->GetWear(WEAR_WEAPON)`

and then uses that current pick for:
- `GetOrePct(ch)`, including current pick refine grade;
- `PracticePick(ch, pick)`, including mastery increment.

`CanHandleItem()` does not block item movement because a mining event is active, and no item equip/unequip path calls `mining_cancel()`.

Therefore a player can start the delayed action with one valid pickaxe, replace it with another pickaxe before completion, and have the second pickaxe determine success chance and receive mastery progress.

Promoted as `BUG-MIN-003`.

## BUG-MIN-004 — direct warp preserves active mining and event does not revalidate map/distance
Normal movement cancels mining through `OnMove() -> mining_cancel()`.

Direct warp follows a different path:
- `CanWarp()` has no `m_pkMiningEvent` check;
- `WarpSet()` calls `Stop()`, not `OnMove()`;
- `Stop()` does not cancel mining;
- `WarpSet()/WarpEnd()` do not explicitly cancel `m_pkMiningEvent`.

The delayed `mining_event` later resolves the original ore by global VID but does not check:
- player map equals ore map;
- current distance to ore;
- current coordinates relative to the original mining point.

On same-process warps where the character object survives and the original vein still exists, the mining result can therefore complete after relocation. `OreDrop()` places the ore at the player's current coordinates/map.

Promoted as `BUG-MIN-004`.

## BUG-MIN-005 — MINING_LOCATION anti-hack branch is unreachable
`CHARACTER::mining()` first rejects:
`map mismatch || distance > 1000`
with an immediate return.

Later it contains:
`if (distance > 2500) { HackLog("MINING_LOCATION"); return; }`

Every distance greater than 2500 is already greater than 1000 and has returned earlier.

Therefore the explicit `MINING_LOCATION` detection/logging branch cannot execute for distance abuse.

Promoted as `BUG-MIN-005`.

## OreRefine ordering closure status
`OreRefine()` still subtracts 100 raw ore before checking whether the player has enough gold:
1. owner/count/refined-vnum checks;
2. `item->SetCount(item->GetCount() - 100)`;
3. compute fee;
4. if gold is insufficient, return false.

The local resource-loss defect is confirmed in the function. However no current deployed quest source in the mapped quest tree has been found calling `pc.ore_refine` or `pc.diamond_refine`. It remains a **reachability candidate**, not yet promoted as a deployed gameplay bug.

## Multiplayer ownership closure
Normal mining drops use 15-second ownership for the miner. Battle Field maps intentionally skip `SetOwnership`. No independent ownership bug is promoted in this pass.


## Skill-book and teardown closure
### Mining skill book
`ITEM_MINING_SKILL_TRAIN_BOOK` routes through `LearnSkillByBook(SKILL_MINING, pct)`.

`LearnSkillByBook` itself rejects reads while `GetSkillNextReadTime(SKILL_MINING)` is still active. On a valid attempt the outer item handler consumes the book and sets the next read time.

No separate cooldown-bypass bug is promoted.

### Logout / character teardown
Final character destruction explicitly calls `event_cancel(&m_pkMiningEvent)`. A normal logout/destruction therefore does not preserve the player mining event.

### Cross-core warp
If a warp causes the source character object to be destroyed, normal teardown cancels the mining event. The verified location-lifecycle defect in `BUG-MIN-004` is limited to same-process/same-character warp paths where the character object survives.

## Current closure position
Verified Mining bugs: `BUG-MIN-001..005`.

Still open before STATIC COMPLETE:
- OreRefine deployed-call reachability;
- compiled quest/object evidence for ore refinement;
- any mining-specific Battle Pass integration expected by current configs;
- final vein/concurrency and pickaxe data sanity pass.


## BUG-MIN-006 — pending mining can settle from a vein already killed by Mining Event shutdown
The custom scheduled Mining Event runs on `EVENT_MAP_INDEX = 230`.

When the event is stopped, `SetMiningEvent(false)`:
1. schedules players to be warped out after 15 seconds;
2. calls `regen_free_map(EVENT_MAP_INDEX)`;
3. runs `FKillSectree`, whose vein branch calls `ch->Dead()`.

For a non-PC, normal `Dead()` does not immediately destroy the character. In the ordinary branch it creates `dead_event` with a 10-second delay.

Player mining attempts are not cancelled by `SetMiningEvent(false)` / `FKillSectree`.

The delayed `mining_event` callback only checks whether the stored vein VID still resolves. It does **not** check `load->IsDead()`.

Therefore, during the dead-vein retention window, a mining attempt that was already in progress can:
- resolve the dead vein by VID;
- use its unchanged race VNUM;
- roll normal mining success;
- drop ore;
- grant pickaxe practice.

Promoted as `BUG-MIN-006`.

## Candidate closures from this pass
- **OreRefine payment ordering:** local C++ ordering is unsafe, but the deployed `guild_building_melt.quest` checks the exact computed gold requirement before calling `pc.ore_refine` / `pc.diamond_refine`. No current insufficient-gold reachability is promoted.
- **Vein VID reuse:** normal VID allocation is monotonically increasing. Destroyed vein VIDs are removed from the manager and are not normally reused during a mining attempt; no stale-VID retarget bug promoted.
- **Battle Field ownership:** map 357 has no deployed vein spawns, and the dynamic Mining Event uses map 230 while the alternate mining initializer uses map 103. The no-ownership Battle Field branch is not currently reachable through mapped mining content.
- **Pickaxe proto rows:** DumpProto contains proto blobs, but the available representation does not yield authoritative readable 29101..29109 rows. No values are inferred.


## BUG-MIN-007 — scheduled Mining Event is deployment-incomplete and enters false-active state
The enabled Event Manager registers `EVENT_TYPE_MINING` and routes it to `SetMiningEvent()`.

The configured event map constant is:
`EVENT_MAP_INDEX = 230`.

Current deployed `Project_Game/share/locale/europe/map/index` contains no map index 230.

`SetMiningEvent(true)` performs operations in this order:
1. `UpdateGameFlag("mining_event", true)`;
2. `SECTREE_MANAGER::GetMap(EVENT_MAP_INDEX)`;
3. if map is missing, return false;
4. otherwise clear regen and load `data/event/mining_event_regen_type_0.txt`.

Therefore in the current deployment:
- the global/event flag is switched to active first;
- map 230 cannot be resolved;
- the start function fails;
- event state can advertise active while no Mining Event map/spawns exist.

Additionally, the referenced regen file `data/event/mining_event_regen_type_0.txt` is absent from the tracked runtime/game repository snapshot, so even adding map 230 alone would not complete this path.

Promoted as `BUG-MIN-007`.

## Final static closure notes
Closed without promotion:
- `OreRefine()` removes ore before its internal gold check, but deployed `guild_building_melt.quest` prechecks the exact gold cost before invoking the Lua API.
- Battle Field ore no-ownership branch is not reachable through current mapped content: map 357 has no vein spawns; scheduled mining event is map 230; alternate mining initializer targets map 103.
- destroyed vein VID retargeting is not a normal risk because character VIDs increment monotonically and destroyed entries are removed from the VID map.
- `InitializeMiningEvent()` also references missing `data/event/mining/map_mining.txt`, but no active caller was established in this snapshot; retained as deployment-risk evidence rather than a separate verified bug.
- no Battle Pass mining mission type or Achievement mining task type exists, so no missing integration hook is expected.
- readable authoritative pickaxe proto rows were not available from the tracked proto blobs; no numeric values were inferred.

## Static closure
Coverage completed for:
- mining start/cancel/event settlement;
- movement/death/warp/logout lifecycle;
- pickaxe identity, mastery and refine quest binding;
- distance/anti-hack validation;
- vein lifetime and VID lifecycle;
- ore drop/ownership behavior;
- OreRefine Lua + deployed quest transaction;
- scheduled Mining Event deployment/config;
- Battle Pass/Achievement integration expectations.

**Mining / Pickaxe: STATIC COMPLETE.**
