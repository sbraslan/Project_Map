# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Static Mapping Coverage Complete  
**Status:** FISHING RENEWAL STATIC COMPLETE / 11 VERIFIED BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** none  
**Latest completed subsystem:** Fishing Renewal  
**System:** `systems/fishing.md`  
**Bugs:** `bugs/fishing.md`  
**Tests:** `tests/fishing.md`  
**Effective completed/readiness-covered subsystems:** 30  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Fishing Renewal closure
Verified bug set:
- `BUG-FISH-001` second normal table OOB selection.
- `BUG-FISH-002` temporary CreateItem(50187) leak.
- `BUG-FISH-003` client-authoritative minigame hit validation.
- `BUG-FISH-004` movement not locked/revalidated.
- `BUG-FISH-005` Carbon rod special bonus branch unreachable.
- `BUG-FISH-006` death/warp lifecycle cleanup missing.
- `BUG-FISH-007` fish_new_log records rerolled VNUM.
- `BUG-FISH-008` renewed success omits Achievement TYPE_FISH hook.
- `BUG-FISH-009` Battle Pass routes renewed catch to FISH_CATCH instead of FISH_FISHING.
- `BUG-FISH-010` active-session logout persists consumed bait socket.
- `BUG-FISH-011` unthrottled CATCH_FAILED packets are immediately rebroadcast to nearby clients.

Closed without promotion:
- `POINT_FISHING_RARE` uint8 narrowing: no current producer/value range proves a reachable overflow in this snapshot.
- failed-counter overflow: theoretical extreme; meaningful reachable issue captured by BUG-FISH-011.

## Next static action
No active subsystem. On the next continuation, choose the next independent unmapped gameplay subsystem from source/config coverage and open only its canonical files.

Do not execute `FISH-T01..FISH-T11`. Global first live runtime gate remains `DUNGEON-T09`.
