# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Horse / Mount / Riding Static Complete  
**Status:** STATIC COMPLETE / 3 PROMOTED HORSE-MOUNT BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Horse / Mount / Riding  
**System:** `systems/horse_mount.md`  
**Bugs:** `bugs/horse_mount.md`  
**Tests:** `tests/horse_mount.md`  
**Last completed subsystem:** Horse / Mount / Riding  
**Effective completed/readiness-covered subsystems:** 28  
**First future live gate:** `DUNGEON-T10`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Horse / Mount final verified set
- `BUG-HORSE-001` — active `horse_ride.quest -> pc.mount()` changes `POINT_MOUNT` without synchronizing `MountVnum`.
- `BUG-HORSE-002` — tracked deployment has horse level/grade consumers but no normal-player horse-level progression producer.
- `BUG-HORSE-003` — current mount items 71259..71266 map to races 20276..20283, but client `GetMountLevelByVnum()` has no cases for them and therefore blocks mounted combat/horse skills.

Deferred tests:
- `HORSE-T01`
- `HORSE-T02`
- `HORSE-T03`

## Final closure notes
- persistence/login/stamina-event lifecycle closed;
- normal mount item/costume equip, unequip, death, warp and expiry cleanup closed;
- client `MountVnum -> actor reinsert -> AdditionalInfo -> MountHorse` render path closed;
- horse-name and horse-appearance persistence closed;
- Achievement SUMMON_MOUNT caller gap remains owned by `BUG-ACH-006`;
- current packed item_proto was decoded and matched between Project_Binary and Project_DumpProto;
- time-limited ChangeLook mount donor lifetime transfer remains unpromoted because no tracked normal-player transmutation opener is proven;
- newer mount raw client-pack MSM/GR2 assets are not versioned, so asset-presence verification is deferred.

No Horse/Mount runtime test has been run.

## Next
Select the next unmapped static subsystem from Project_Map and continue detection-only mapping.

GitHub state is canonical.
