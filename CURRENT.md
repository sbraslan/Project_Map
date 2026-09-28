# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Horse / Mount / Riding Static Mapping  
**Status:** STATIC MAPPING IN PROGRESS / 2 PROMOTED HORSE-MOUNT BUGS / EXECUTION LOCKED  
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

## Horse / Mount verified bugs
- `BUG-HORSE-001` — active `horse_ride.quest -> pc.mount()` changes `POINT_MOUNT` but does not synchronize `MountVnum`.
- `BUG-HORSE-002` — tracked quest deployment has horse level/grade consumers but no normal-player horse-level progression producer.

Deferred tests:
- `HORSE-T01` — POINT_MOUNT / MountVnum parity.
- `HORSE-T02` — zero-level horse progression reachability.

## Closed this turn
- real-time/timer-on-wear mount expiry reaches `RemoveFromCharacter -> Unequip` and synchronizes mount cleanup; no stale-expiry bug.
- Achievement `TYPE_SUMMON_MOUNT` gap is already owned by `BUG-ACH-006`; no duplicate Horse bug.
- ChangeLook mount lifetime transfer remains a strong candidate, but current connector cannot prove a concrete time-limited COSTUME_MOUNT row from the encoded DumpProto item_proto snapshot, so it remains unpromoted.

## Exact next work
1. close Additional Equipment Page interaction with UNIQUE ride items;
2. close client mount packet/race/assets and horse appearance;
3. audit horse-name/appearance persistence and ChangeLook interaction;
4. decide Horse/Mount STATIC COMPLETE.

Do not execute `HORSE-T01` or `HORSE-T02`.
GitHub state is canonical.
