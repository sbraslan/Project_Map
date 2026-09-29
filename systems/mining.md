# Mining / Pickaxe

**Status:** MAPPING IN PROGRESS / 2 VERIFIED BUGS / EXECUTION LOCKED  
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
