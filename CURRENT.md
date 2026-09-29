# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** 6th/7th Attribute static mapping  
**Status:** ATTR6TH7TH — STATIC MAPPING OPEN / 8 VERIFIED BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** 6th/7th Attribute — OPEN  
**System:** `systems/attr6th7th.md`  
**Bugs:** `bugs/attr6th7th.md`  
**Tests:** `tests/attr6th7th.md`  
**Previous completed subsystem:** Monarch — STATIC COMPLETE / 13 VERIFIED BUGS  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Recent static closure
- Monarch is closed with `BUG-MON-001..013`.
- Deferred Monarch tests are `MON-T01..MON-T13`; none executed.
- Attr67 mapping has verified `BUG-ATTR67-001..008`; runtime remains locked.
- Context7 supplied supplementary C++/MySQL semantics checks only.

## Exact resume cursor
1. Finish Attr67 private-shop/shop/material cross-window boundary pass.
2. Audit NPC_STORAGE delayed retrieval plus reconnect/restart reconstruction.
3. Audit percent/support bounds and rare-attribute mutation.
4. Close deployment-shadowed client/server parity candidates.
5. Decide STATIC COMPLETE; source/game repos stay read-only and runtime stays locked.

## Mapping acceleration index
- Status: **READY / ACTIVE**
- Indexed mapping-relevant files: **10,119**
- Primary lookup: `features -> symbols/packets -> callgraph -> files -> exact source fetch`.
- Context7: supplementary external semantics verification; Metin2 source remains authoritative.
