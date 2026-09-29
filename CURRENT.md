# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** OX Event static mapping  
**Status:** OX EVENT — STATIC MAPPING IN PROGRESS / 0 VERIFIED BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** OX Event — OPEN  
**System:** `systems/oxevent.md`  
**Bugs:** `bugs/oxevent.md`  
**Tests:** `tests/oxevent.md`  
**Previous completed subsystem:** Marriage / Wedding — STATIC COMPLETE / 8 VERIFIED BUGS  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Latest closure
- Marriage/Wedding is statically complete with `BUG-MARR-001..008`.
- Deferred Marriage tests are `MARR-T01..MARR-T08`; none has been executed.
- Major closure findings include engagement transaction ordering, duplicate map-81 producers, divorce fee boundary, stale P2P lover state, wedding exit/membership lifecycle, EXP love-point truncation, and cross-core marriage-item sharing.

## Exact resume cursor
1. Close client lover command/UI lifecycle and repeated LoverInfo behavior.
2. Close remaining Lua null/state assumptions against deployed callers.
3. Close marriage-bonus consumers and wedding teardown edges.
4. If no new independent finding is proven, mark Marriage/Wedding STATIC COMPLETE and select the next indexed subsystem.
5. Keep source/game repositories read-only and runtime execution locked.

## Mapping acceleration index
- Status: **READY / ACTIVE**
- Indexed mapping-relevant files: **10,119**
- Primary lookup: `index/features.json -> symbols/packets -> callgraph -> files.json -> exact source fetch`.
- Context7 remains supplementary for external APIs only.
