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

## Growth Pet closure

Growth Pet System is now **STATIC COMPLETE**.

Canonical verified findings:
`BUG-GPET-001..BUG-GPET-020`.

Canonical deferred tests:
`GPET-T01..GPET-T20`, none executed.

Closure includes:
- hatch-info, feed packet and skill-slot array boundaries;
- skill-book/type/VNUM validation;
- premium revive validation/persistence;
- evolution material identity;
- specialist/passive/active skill execution;
- packet-name boundaries;
- birth/age/evolution semantics;
- attribute-change state ownership;
- multi-slot feed behavior;
- separate Growth Pet DB row lifecycle;
- summon/dismiss/death/real-time expiry;
- level 1..105 EXP arithmetic;
- same-core/cross-core pet actor lifetime;
- current 55701..55713 family/data coverage;
- PET_BAG lifecycle.

Final added findings:
- `BUG-GPET-019` — transport-box bagging destroys the target pet seal and then dereferences `item2->GetName()`;
- `BUG-GPET-020` — PET_BAG accepts a dead Growth Pet and unbagging recreates it with `now + pet_max_time`, bypassing the intended revive path.

Current effective coverage is **27/27 STATIC COMPLETE subsystem rows** plus folded Guild lifecycle.

## Active Horse / Mount / Riding state

Initial mapped roots:
- `horse_rider.cpp/.h`;
- `char_horse.cpp`;
- `questlua_horse.cpp`;
- mount/ride item paths in `char_item.cpp` and `item.cpp`;
- mount state/stat integration in `char.cpp`.

Initial lifecycle:
- classic horse health/stamina/ride events;
- horse actor summon/dismiss/death/revive;
- character ride/unride -> `MountVnum`;
- UNIQUE/COSTUME mount item layer;
- mount ChangeLook expiry helper surface.

No Horse/Mount-specific bug is promoted yet.

Active candidate:
- `CItem::IsExpireTimeItem()` uses a broad type/subtype predicate, but no live caller/consequence has yet been closed.

## Exact next work
1. close horse persistence/login/logout and horse-level bounds;
2. audit stamina/health events and repeated ride-state transitions;
3. audit quest horse API authorization/range handling;
4. map mount item/costume -> affect -> `MountVnum` lifecycle;
5. audit expiry/unequip/death/warp cleanup;
6. close ChangeLook mount expiry caller graph;
7. resolve Achievement SUMMON_MOUNT producer gap;
8. close client race/proto/horse-appearance coverage;
9. promote only verified reachable Horse/Mount bugs and tests.

Do not execute any Horse/Mount runtime test.
The global future runtime order remains locked with `DUNGEON-T10` first.

GitHub state is canonical.
