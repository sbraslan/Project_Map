# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Arena / PvP Duel static mapping  
**Status:** ARENA — STATIC MAPPING IN PROGRESS / 1 VERIFIED BUG / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Arena / PvP Duel — OPEN  
**System:** `systems/arena.md`  
**Bugs:** `bugs/arena.md`  
**Tests:** `tests/arena.md`  
**Previous completed subsystem:** OX Event — STATIC COMPLETE / 5 VERIFIED BUGS  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Latest closures
- Marriage/Wedding: STATIC COMPLETE / `BUG-MARR-001..008`.
- OX Event: STATIC COMPLETE / `BUG-OX-001..005`.
- OX deferred tests: `OX-T01..OX-T05`; none executed.

## Active Arena finding
- `BUG-ARENA-001` — deployed `arena.is_in_arena` contract is inverted relative to the quest: normal idle eligible opponents return 0 and are rejected before `arena.start_duel`.
- Shadowed lower-layer findings: map112 has no tracked MAP_ALLOW owner; timeout packet is sent to A twice/not B; observer teardown needs post-deployment closure.
- Deferred canonical test: `ARENA-T01`; not executed.

## Exact resume cursor
1. OX Event is statically closed with `BUG-OX-001..008`.
2. Open Arena as the next indexed subsystem.
3. Map duel start/timeout/death/observer/disconnect lifecycles.
4. Keep source/game repositories read-only and runtime execution locked.

## Mapping acceleration index
- Status: **READY / ACTIVE**
- Indexed mapping-relevant files: **10,119**
- Primary lookup: `features -> symbols/packets -> callgraph -> files -> exact source fetch`.
- Context7 remains supplementary for external APIs only.
