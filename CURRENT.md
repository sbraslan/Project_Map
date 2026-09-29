# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Mining / Pickaxe mapping in progress  
**Status:** 30 STATIC COMPLETE / MINING ACTIVE / 5 VERIFIED BUGS / EXECUTION LOCKED  
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
- `BUG-MIN-003` — delayed mining is not bound to the initiating pickaxe; a replacement pick controls final chance and receives mastery.
- `BUG-MIN-004` — direct same-process warp can preserve mining; completion does not revalidate map/distance and can drop ore at destination.
- `BUG-MIN-005` — `MINING_LOCATION` >2500 HackLog branch is unreachable because >1000 returns first.

## Exact next work
1. close OreRefine resource/payment ordering against deployed/compiled quest reachability;
2. audit mining skill-book delay/skill progression integration;
3. audit vein spawn/despawn and concurrent-player contention;
4. inspect Battle Pass/Achievement hooks for mining/ore actions;
5. close logout/disconnect and cross-core warp behavior;
6. cross-check pickaxe progression/proto values if an authoritative non-empty proto source becomes available;
7. promote only statically verified reachable findings.

Do not execute `MIN-T01`..`MIN-T05`. Global first live runtime gate remains `DUNGEON-T09`.
