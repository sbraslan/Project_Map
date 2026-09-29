# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Static Mapping Coverage Complete  
**Status:** MINING / PICKAXE STATIC COMPLETE / 6 VERIFIED BUGS / EXECUTION LOCKED  
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
- `BUG-MIN-001` — deployed mining quest and C++ disagree on pickaxe mastery refine boundary.
- `BUG-MIN-002` — delayed mining can resolve ore/mastery after player death.
- `BUG-MIN-003` — same-process warp can carry active mining across maps and resolve remotely.
- `BUG-MIN-004` — event settlement uses the currently equipped pickaxe rather than the initiating pickaxe.
- `BUG-MIN-005` — `MINING_LOCATION` >2500 anti-hack branch is unreachable behind an earlier >1000 return.
- `BUG-MIN-006` — scheduled Event Manager Mining Event targets missing map 230 / missing regen deployment data.

Closed without promotion:
- `OreRefine()` removes raw ore before its internal gold check, but deployed `guild_building_melt.quest` checks the same fee first.
- Battle Field map 357 has no deployed mining veins; ownership exception is not currently reachable through tracked mining data.
- vein despawn cleanly resolves to missing VID and aborts; no practical VID reuse alias established.
- alternate map-103 `InitializeMiningEvent()` path is defined but no active caller was identified in the tracked server snapshot.
- pickaxe item_proto rows are not available as readable text in the tracked dump artifact, but item names confirm VNUMs 29101..29109 as +0..+8.

## Next static action
No active subsystem. On the next continuation, select the next independent unmapped gameplay subsystem from source/config coverage and open only its canonical files.

Do not execute `MIN-T01..MIN-T06`. Global first live runtime gate remains `DUNGEON-T09`.


## Pause checkpoint
Mapping was intentionally paused by the user because the conversation/tool flow was repeatedly interrupted.

Resume from **Mining / Pickaxe** without redoing completed work.

### Saved Mining state
- `BUG-MIN-001` — deployed pickaxe refine quest/C++ mastery boundary mismatch.
- `BUG-MIN-002` — delayed mining can resolve after death.
- `BUG-MIN-003` — same-process warp can carry active mining across maps.
- `BUG-MIN-004` — settlement uses the currently equipped pickaxe, not the initiating pickaxe.
- `BUG-MIN-005` — `MINING_LOCATION` >2500 anti-hack branch is unreachable due to earlier >1000 return.

### Closed without promotion
- `OreRefine()` removes 100 ore before its internal gold check, but deployed `guild_building_melt.quest` performs the same affordability check immediately before calling `pc.ore_refine/pc.diamond_refine`; no deployed insufficient-gold loss path is currently proven.
- vein VID reuse was not promoted: character VIDs are monotonically allocated in the observed process and destroyed veins are removed from the VID map.
- Battle Field map 357 static data has no mining veins in `regen.txt`, `npc.txt` or `boss.txt`; `stone.txt` contains non-mining stones. Dynamic Mining Event spawn audit was **started but not completed**.
- Pickaxe proto/value cross-check remains open because the available DumpProto `item_proto` blob was not readable through the connector.

### Exact resume cursor
1. Finish `ENABLE_MINING_EVENT` dynamic spawn audit, especially whether veins can spawn on Battle Field map 357.
2. Close Battle Field ore ownership reachability.
3. Close pickaxe proto/value progression if an authoritative readable proto source is available.
4. Reassess whether Mining can be marked STATIC COMPLETE.
5. Do not execute runtime tests; global first future live gate remains `DUNGEON-T09`.
