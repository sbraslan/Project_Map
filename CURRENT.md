# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Mining / Pickaxe mapping in progress  
**Status:** 30 STATIC COMPLETE / MINING ACTIVE / 2 VERIFIED BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Mining / Pickaxe  
**System:** `systems/mining.md`  
**Bugs:** `bugs/mining.md`  
**Tests:** `tests/mining.md`  
**Last completed subsystem:** Fishing Renewal  
**Effective completed/readiness-covered subsystems:** 30  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Verified Mining findings
- `BUG-MINE-001` — current mining quest requires pick mastery `socket0 == value2`, while C++ `Pick_Refinable` rejects equality and requires `socket0 > value2`; the normal quest refine path has no satisfiable state.
- `BUG-MINE-002` — delayed mining event is not cancelled/revalidated on death or warp and can resolve ore at dead/destination state.

## Open candidates
- `OreRefine` removes 100 raw ore before checking Yang affordability; current Project_Game caller not yet established.
- the later >2500 `MINING_LOCATION` hack-log branch is unreachable because an earlier >1000 check already returns.

## Exact next work
1. map click/quest entry ownership and packet trust boundary;
2. audit logout/equipment-change and delayed-event cleanup;
3. audit pick mastery/refine extended-inventory cell behavior;
4. establish current OreRefine caller/reachability and fee/material atomicity;
5. audit mining skill-book progression/cooldown;
6. audit ore ownership/Battle Field/multiplayer interactions;
7. promote only verified reachable findings.

Do not execute `MINE-T01` or `MINE-T02`. Global first live runtime gate remains `DUNGEON-T09`.
