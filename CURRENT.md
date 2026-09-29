# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** 6th/7th Attribute static mapping  
**Status:** ATTR6TH7TH — STATIC MAPPING OPEN / 0 VERIFIED BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** 6th/7th Attribute — OPEN  
**System:** `systems/attr6th7th.md`  
**Bugs:** `bugs/attr6th7th.md`  
**Tests:** `tests/attr6th7th.md`  
**Previous completed subsystem:** Monarch — STATIC COMPLETE / 11 VERIFIED BUGS  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Recent static closure
- Monarch is closed with `BUG-MON-001..011`.
- Deferred Monarch tests are `MON-T01..MON-T11`; none executed.
- Final Monarch finding `BUG-MON-011`: local `mto` and cross-core `mtr` lose private-instance map identity.
- Context7 supplied supplementary C++/MySQL semantics checks only.

## Exact resume cursor
1. Open `Project_ServerSRC/game/src/Attr6th7th.cpp/.h`.
2. Map `questlua_attr6th7th.cpp` bindings and deployed quest callers.
3. Trace item/material/cost mutation ordering and ownership/state revalidation.
4. Trace any client/server packet/UI integration.
5. Keep source/game repositories read-only and runtime execution locked.

## Mapping acceleration index
- Status: **READY / ACTIVE**
- Indexed mapping-relevant files: **10,119**
- Primary lookup: `features -> symbols/packets -> callgraph -> files -> exact source fetch`.
- Context7: supplementary external semantics verification; Metin2 source remains authoritative.
