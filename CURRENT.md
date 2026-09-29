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
- `BUG-MIN-001` — deployed mining quest requires pickaxe mastery socket0 == Value2, while C++ `Pick_Refinable()` accepts only socket0 > Value2; normal refine path is contradictory.
- `BUG-MIN-002` — delayed mining event is not cancelled on death and does not check `IsDead()`, so ore/mastery can resolve after death.

## Exact next work
1. close direct warp/map-change cleanup and distance revalidation;
2. audit equipment swap/unequip during the delayed event;
3. close the unreachable >2500 MINING_LOCATION anti-hack branch;
4. audit OreRefine resource/payment ordering and current quest reachability;
5. audit ore ownership/Battle Field and multiplayer contention;
6. cross-check pickaxe progression/proto values where authoritative rows exist;
7. promote only statically verified reachable findings.

Do not execute `MIN-T01` or `MIN-T02`. Global first live runtime gate remains `DUNGEON-T09`.
