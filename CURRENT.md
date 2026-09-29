# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Monarch static mapping  
**Status:** MONARCH — STATIC MAPPING IN PROGRESS / 9 VERIFIED BUGS / EXECUTION LOCKED  
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
- `BUG-MON-003` — `DelMonarch` checks DELETE via `uiNumRows` instead of `uiAffectedRows`; persistent deletion can occur while runtime state remains stale, and RMMonarch calls it twice.
- `BUG-MON-004` — `mtax` reports values outside 1..50 as invalid but still writes the invalid `trade_tax` flag and cooldown.
- `BUG-MON-005` — different monarch actions can pass the same stale treasury balance before async DB deduction returns, producing an effect DB later cannot charge.
- `BUG-MON-006` — MI_TAX is set but never checked, so tax cooldown is unenforced.
- `BUG-MON-007` — remote `mtr` charges treasury/cooldown before any delivery acknowledgement; target disappearance makes it fail silently.
- `BUG-MON-008` — DB restart reloads only monarch identity/treasury, not persisted candidacy/vote runtime state.
- `BUG-MON-009` — new character sessions run `InitMC()` and reset enforced Monarch cooldowns to immediately ready.
- Deferred tests: `MON-T01..MON-T09`; none executed.

## Recent static closures
- Arena / PvP Duel: `BUG-ARENA-001..003`.
- OX Event: canonical registry remains closed; runtime locked.
- Marriage / Wedding: canonical registry remains closed; runtime locked.

## Exact resume cursor
1. Close PowerUp/DefenseUp and `takemonarchmoney` as deployed versus dormant.
2. Audit add-money overflow/failure reporting symmetry.
3. Audit remaining monarch notice and warp boundaries.
4. Decide Monarch STATIC COMPLETE and select the next subsystem.
5. Keep source/game repositories read-only and runtime execution locked.

## Mapping acceleration index
- Status: **READY / ACTIVE**
- Indexed mapping-relevant files: **10,119**
- Primary lookup: `features -> symbols/packets -> callgraph -> files -> exact source fetch`.
- Context7: supplementary C++/library semantics verification; Metin2 source remains authoritative.
