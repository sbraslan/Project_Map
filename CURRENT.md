# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Global source-feature coverage discovery  
**Status:** 37 STATIC COMPLETE SUBSYSTEMS / GLOBAL DISCOVERY OPEN / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Navigation:** `INDEX.md`  
**Runtime readiness:** `RUNTIME.md`  
**Previous completed subsystem:** 6th/7th Attribute — STATIC COMPLETE / 9 VERIFIED BUGS  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Recent static closure
- Attr67 closed with `BUG-ATTR67-001..009`.
- Deferred Attr67 tests are `ATTR67-T01..ATTR67-T09`; none executed.
- NPC_STORAGE + quest-flag persistence was mapped without a new reconnect/restart loss bug.
- Remaining Attr67 retrieval hazards are dormant because the tracked deployment has no retrieval caller.
- All **37 subsystem rows currently represented in `INDEX.md` are STATIC COMPLETE**; Guild lifecycle remains folded with Guild Storage.
- Context7 remains supplementary; Metin2 source is authoritative.

## Exact resume cursor
1. Run global source-feature coverage discovery against the indexed source corpus.
2. Compare feature flags, packet families, quest/Lua bindings, server managers and client UI/modules against `INDEX.md`.
3. If an unmapped feature family is found, open exactly one new subsystem and continue static mapping.
4. If no unmapped family is found, record global static coverage closure/readiness handoff.
5. Keep source/game repos read-only and runtime locked.

## Mapping acceleration index
- Status: **READY / ACTIVE**
- Indexed mapping-relevant files: **10,119**
- Primary lookup: `features -> symbols/packets -> callgraph -> files -> exact source fetch`.
- Context7: supplementary external semantics verification only.
