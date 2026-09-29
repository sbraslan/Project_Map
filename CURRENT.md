# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Monarch static mapping  
**Status:** MONARCH — STATIC MAPPING IN PROGRESS / 4 VERIFIED BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Monarch — OPEN  
**System:** `systems/monarch.md`  
**Bugs:** `bugs/monarch.md`  
**Tests:** `tests/monarch.md`  
**Previous completed subsystem:** Arena / PvP Duel — STATIC COMPLETE / 3 VERIFIED BUGS  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Active Monarch findings
- `BUG-MON-001` — registered election finalizer never selects/persists/publishes a winner; internal count uses voter PID and uninitialized scalar counters.
- `BUG-MON-002` — `setmonarch` persists DB state but sends the wrong DG header and never refreshes live game-core monarch info.
- `BUG-MON-003` — `rmmonarch` calls deletion twice, so a successful first removal cannot produce a successful second result/fanout.
- `BUG-MON-004` — `mtax` reports values outside 1..50 as invalid but still writes the invalid `trade_tax` flag and cooldown.
- Deferred tests: `MON-T01..MON-T04`; none executed.

## Recent static closures
- Arena / PvP Duel: `BUG-ARENA-001..003`.
- OX Event: canonical registry remains closed; runtime locked.
- Marriage / Wedding: canonical registry remains closed; runtime locked.

## Exact resume cursor
1. Close treasury request/ack concurrency and failed-deduction semantics.
2. Audit `takemonarchmoney` authorization/caller reachability.
3. Audit process-local PowerUp/DefenseUp cross-core behavior and deployed callers.
4. Audit monarch warp/transfer charge-on-failure ordering.
5. Keep source/game repositories read-only and runtime execution locked.

## Mapping acceleration index
- Status: **READY / ACTIVE**
- Indexed mapping-relevant files: **10,119**
- Primary lookup: `features -> symbols/packets -> callgraph -> files -> exact source fetch`.
- Context7: supplementary C++/library semantics verification; Metin2 source remains authoritative.
