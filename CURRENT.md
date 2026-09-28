# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Horse / Mount / Riding Static Mapping  
**Status:** STATIC MAPPING IN PROGRESS / 0 PROMOTED HORSE-MOUNT BUGS / EXECUTION LOCKED  
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
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime, crafted-packet, crash, sanitizer and fault-injection execution remains locked until an explicit phase change.

## Horse / Mount progress this turn
Closed statically:
- horse DB save/load for level/riding/HP/stamina/health-drop time;
- login riding-state reconstruction through `EnterHorse()`;
- level setter clamp `0..30`;
- stamina consume/regen event ownership and destructor cleanup;
- current-build classification: `ENABLE_INFINITE_HORSE_HEALTH_STAMINA` is enabled.

No Horse/Mount bug was promoted.

Important candidates kept open:
- DB-loaded horse level is not clamped before horse-stat table use, but no tracked malformed producer exists.
- `StartChangeLookExpireEvent()` accepts costume mounts, while automatic start sites currently found only trigger it for horse-summon items; socket2 producer/reachability still needs closure.
- login `SetHorseLevel(GetHorseLevel())` resets underlying HP/stamina/drop-time, but infinite horse health/stamina masks a distinct current consequence.

## Exact next work
1. audit quest horse API authorization/range handling and quest producers;
2. map mount item/costume -> affect -> `MountVnum`;
3. audit expiry/unequip/death/warp cleanup;
4. close ChangeLook mount socket2 producer/caller graph;
5. resolve Achievement SUMMON_MOUNT producer gap;
6. close client race/proto/appearance coverage;
7. promote only verified reachable Horse/Mount bugs/tests.

Do not execute any Horse/Mount runtime test.
GitHub state is canonical.
