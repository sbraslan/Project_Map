# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Monarch static mapping  
**Status:** MONARCH — STATIC MAPPING IN PROGRESS / 10 VERIFIED BUGS / EXECUTION LOCKED  
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
- `BUG-MON-001` — election finalizer never selects/persists/publishes a winner and its counter logic is invalid.
- `BUG-MON-002` — `setmonarch` does not refresh live game-core monarch state.
- `BUG-MON-003` — `rmmonarch` DELETE result handling uses result rows instead of affected rows and leaves stale runtime state.
- `BUG-MON-004` — invalid `mtax` values are reported but still applied.
- `BUG-MON-005` — asynchronous treasury prechecks can grant an effect the DB cannot ultimately charge.
- `BUG-MON-006` — legacy `setmonarch` persistence writes the selected PID through `name` while authoritative reload reads `pid`.
- `BUG-MON-007` — remote `mtr` charges treasury/cooldown without delivery acknowledgement.
- `BUG-MON-008` — DB restart loses candidacy/vote runtime state although election rows were persisted.
- `BUG-MON-009` — relog/CHARACTER recreation resets enforced monarch cooldowns to immediately ready.
- `BUG-MON-010` — `MI_TAX` is written but never checked, so tax cooldown is unenforced.
- Deferred tests: `MON-T01..MON-T10`; none executed.

## Recent static closures
- Arena / PvP Duel: `BUG-ARENA-001..003`.
- OX Event: canonical registry remains closed; runtime locked.
- Marriage / Wedding: canonical registry remains closed; runtime locked.

## Exact resume cursor
1. Audit registered `mto/mtr` private-instance and map-index semantics.
2. Close remaining `WarpSet` result handling and AddMoney boundaries.
3. Keep Lua-only PowerUp/DefenseUp/takemonarchmoney dormant unless a deployed caller is proven.
4. Decide Monarch STATIC COMPLETE and select the next subsystem.
5. Keep source/game repositories read-only and runtime execution locked.

## Mapping acceleration index
- Status: **READY / ACTIVE**
- Indexed mapping-relevant files: **10,119**
- Primary lookup: `features -> symbols/packets -> callgraph -> files -> exact source fetch`.
- Context7: supplementary C++/library semantics verification; Metin2 source remains authoritative.
