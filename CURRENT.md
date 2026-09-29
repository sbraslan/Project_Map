# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Classic Pet System Static Mapping  
**Status:** STATIC MAPPING IN PROGRESS / 4 PROMOTED CLASSIC-PET BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Classic Pet System  
**System:** `systems/pet.md`  
**Bugs:** `bugs/pet.md`  
**Tests:** `tests/pet.md`  
**Last completed subsystem:** Horse / Mount / Riding  
**Effective completed/readiness-covered subsystems:** 28  
**First future live gate:** `DUNGEON-T10`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Classic Pet progress
Promoted:
- `BUG-PET-001` — Bruce pickup range ignores Y distance.
- `BUG-PET-002` — Bruce caches a raw ground-item `LPITEM` across update ticks; owner manual pickup can destroy the object before the pet dereferences it again.
- `BUG-PET-003` — REAL_TIME PET_PAY expiry removes the summon item without PetUnsummon; the missing-item actor branch returns before Unsummon and the update event keeps scheduling.
- `BUG-PET-004` — forced summon-item loss prevents elapsed TYPE_SUMMON_PET time from being credited; logout later clears the stale summon-time flag without credit.

Deferred tests:
- `PET-T01`
- `PET-T02`
- `PET-T03`
- `PET-T04`

Also closed:
- normal PET_PAY toggle uses explicit `PetUnsummon`;
- forced lower-level item removal does not call `PetUnsummon`;
- missing summon-item branch in `CPetActor::Update` returns false before `Unsummon`;
- pet-system event ignores that Update return and continues scheduling.

Newly closed:
- deployed PET_PAY REAL_TIME reachability is confirmed from current decoded item proto, including Bruce 53233;
- `petsystem_update_event` ignores the false update result and reschedules;
- owner destruction eventually tears down the pet system and stale actor.

## Exact next work
1. locate/exclude active `pet.summon()` quest producers;
2. map remaining PET_PAY item/race/client coverage;
3. close Achievement TYPE_SUMMON_PET abnormal cleanup consequence;
4. audit remaining death/warp/login restoration edges.

Do not execute `PET-T01`, `PET-T02` or `PET-T03`.
GitHub state is canonical.


## Latest checkpoint
- active tracked quest package contains no `pet.summon()`/related producer; legacy Lua signature mismatch remains dormant and is not promoted;
- Achievement 60 actively tracks 30 days of `TYPE_SUMMON_PET` time;
- REAL_TIME item loss can discard the current uncommitted summon interval -> `BUG-PET-004`.

## Exact next work
1. finish PET_PAY item/race/client coverage;
2. close death/warp/login restoration edges;
3. audit remaining auto-pickup ownership/re-target lifetime boundaries;
4. decide Classic Pet STATIC COMPLETE.

Do not execute `PET-T01`..`PET-T04`.
