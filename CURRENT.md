# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Classic Pet System Static Mapping  
**Status:** STATIC MAPPING IN PROGRESS / 2 PROMOTED CLASSIC-PET BUGS / EXECUTION LOCKED  
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

Deferred tests:
- `PET-T01`
- `PET-T02`

Also closed:
- normal PET_PAY toggle uses explicit `PetUnsummon`;
- forced lower-level item removal does not call `PetUnsummon`;
- missing summon-item branch in `CPetActor::Update` returns false before `Unsummon`;
- pet-system event ignores that Update return and continues scheduling.

Strong open candidate:
- forced PET_PAY expiry/removal can leave the pet actor summoned after its item disappears. Concrete current expiry producer/data still needs closure before promotion.

## Exact next work
1. close current PET_PAY expiry reachability;
2. audit CPetSystem event/actor-map cleanup and logout/destruction;
3. locate/exclude active `pet.summon()` quest producers;
4. map PET_PAY item/race/client coverage;
5. audit Achievement TYPE_SUMMON_PET abnormal cleanup.

Do not execute `PET-T01` or `PET-T02`.
GitHub state is canonical.
