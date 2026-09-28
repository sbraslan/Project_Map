# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Horse / Mount / Riding Static Mapping  
**Status:** STATIC MAPPING IN PROGRESS / 1 PROMOTED HORSE-MOUNT BUG / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Horse / Mount / Riding  
**System:** `systems/horse_mount.md`  
**Bugs:** `bugs/horse_mount.md`  
**Tests:** `tests/horse_mount.md`  
**Last completed subsystem:** Growth Pet System  
**Effective completed/readiness-covered subsystems:** 27  
**First future live gate:** `DUNGEON-T10`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Horse / Mount progress this turn
Promoted:
- `BUG-HORSE-001` — active `horse_ride.quest -> pc.mount()` changes `POINT_MOUNT` through an affect but does not synchronize `MountVnum`; `PointChange(POINT_MOUNT)` has its `MountVnum(val)` call commented out.

Deferred test:
- `HORSE-T01` — POINT_MOUNT / MountVnum / client-render parity.

Also closed:
- eight current h_horse quest sources are active in `quest_list`;
- normal equipped mount item/costume add/remove explicitly synchronizes `MountVnum`;
- death cleanup covers special ride unique items and costume mount;
- restricted-map post-warp EnterGame calls `Unmount()`.

Strong open candidate:
- mount ChangeLook transaction stores only donor VNUM and destroys donor; lifetime/socket2/event transfer is missing. Verify a deployed time-limited COSTUME_MOUNT before promotion.

Deployment candidate:
- active h_horse scripts use horse-level gates but contain no horse-level advancement producer.

## Exact next work
1. verify deployed time-limited COSTUME_MOUNT data and close ChangeLook lifetime candidate;
2. locate/exclude non-h_horse horse-level progression producers;
3. close real-time/timer-based mount expiry;
4. resolve Achievement SUMMON_MOUNT producer gap;
5. close client race/proto/horse-appearance coverage.

Do not execute `HORSE-T01`.
GitHub state is canonical.
