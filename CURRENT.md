# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** OX Event static mapping  
**Status:** OX EVENT — STATIC MAPPING IN PROGRESS / 5 VERIFIED BUGS / EXECUTION LOCKED  
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

## Active OX findings
- `BUG-OX-001` — renewal inner timer and automatic 35-second outer scheduler collide deterministically, corrupting alternating quiz stages.
- `BUG-OX-002` — async/equality-only login counter is not an authoritative player cap and can diverge from unique attendees.
- `BUG-OX-003` — an in-flight OPEN warp can be admitted after state changed to CLOSE/QUIZ.
- `BUG-OX-004` — automatic round restart does not reset the deployed admission counter.
- `BUG-OX-005` — deployed GM force-end does not cancel the automatic Event Manager scheduler.
- Deferred tests: `OX-T01..OX-T05`; none executed.

## Latest closure
- Marriage/Wedding is statically complete with `BUG-MARR-001..008`.
- Deferred Marriage tests are `MARR-T01..MARR-T08`; none has been executed.
- Major closure findings include engagement transaction ordering, duplicate map-81 producers, divorce fee boundary, stale P2P lover state, wedding exit/membership lifecycle, EXP love-point truncation, and cross-core marriage-item sharing.

## Exact resume cursor
1. Audit eliminated-player Show()/audience and logout/relog lifecycle.
2. Audit winner reward/offline handling.
3. Close automatic-start dependency on persisted admission flags.
4. Decide OX Event STATIC COMPLETE readiness.
5. Keep source/game repositories read-only and runtime locked.

## Mapping acceleration index
- Status: **READY / ACTIVE**
- Indexed mapping-relevant files: **10,119**
- Primary lookup: `index/features.json -> symbols/packets -> callgraph -> files.json -> exact source fetch`.
- Context7 remains supplementary for external APIs only.
