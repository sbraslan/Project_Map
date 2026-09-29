# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Static Mapping Coverage Complete  
**Status:** MINING / PICKAXE STATIC COMPLETE / 7 VERIFIED BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** none  
**Latest completed subsystem:** Mining / Pickaxe  
**System:** `systems/mining.md`  
**Bugs:** `bugs/mining.md`  
**Tests:** `tests/mining.md`  
**Effective completed/readiness-covered subsystems:** 31  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Mining / Pickaxe closure
Verified bug set:
- `BUG-MIN-001` — deployed pickaxe refine quest and C++ disagree on mastery boundary.
- `BUG-MIN-002` — delayed mining can resolve after player death.
- `BUG-MIN-003` — same-process warp can carry active mining across maps and settle at destination.
- `BUG-MIN-004` — delayed settlement uses the currently equipped pickaxe instead of the initiating pickaxe.
- `BUG-MIN-005` — `MINING_LOCATION` >2500 anti-hack log is unreachable behind the earlier >1000 return.
- `BUG-MIN-006` — scheduled Mining Event shutdown leaves dead veins manager-resolvable long enough for pending mining settlement.
- `BUG-MIN-007` — scheduled Mining Event uses missing map 230 / missing regen data and sets `mining_event` active before start failure.

Closed without promotion:
- OreRefine internal payment ordering is shielded by deployed quest precheck.
- Battle Field no-ownership branch has no mapped vein producer on map 357.
- normal destroyed-vein VID reuse is not reachable under monotonic VID allocation.
- alternate `InitializeMiningEvent()` missing `data/event/mining/map_mining.txt` lacks a proven active caller.
- Battle Pass/Achievement define no mining mission/task type.
- authoritative readable pickaxe proto rows unavailable; no numeric proto values inferred.

## Next static action
No active subsystem. On the next continuation, select the next independent unmapped gameplay subsystem from source/config coverage and open only its canonical files.

Do not execute `MIN-T01..MIN-T07`. Global first live runtime gate remains `DUNGEON-T09`.
