# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Growth Pet System Static Mapping  
**Status:** STATIC MAPPING IN PROGRESS / 1 VERIFIED STATIC BUG / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Growth Pet System  
**System:** `systems/growth_pet.md`  
**Bugs:** `bugs/growth_pet.md`  
**Tests:** `tests/growth_pet.md`  
**Last completed subsystem:** Aura System  
**Effective completed/readiness-covered subsystems:** 25  
**First future live gate:** `DUNGEON-T10`  
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime, crafted-packet, crash, sanitizer and fault-injection execution remains locked until an explicit phase change.

## Aura System closure
Aura is now **STATIC COMPLETE**.

Canonical verified findings:
`BUG-AURA-001..BUG-AURA-006`.

Canonical deferred ownership:
`AURA-T01..AURA-T06`, none executed.

Closure covered:
- packet/state authorization;
- ABSORB / GROWTH / EVOLVE;
- booster/eraser timing and stat application;
- Yohara random-apply persistence;
- terminal Radiant arithmetic;
- packet initialization;
- normal disconnect/lock cleanup;
- warp/opener boundaries;
- current proto/refine families;
- PART_AURA -> MSE visual chain.

Current effective readiness coverage is **25/25 STATIC COMPLETE subsystems**, plus folded Guild lifecycle.

## Active Growth Pet state
First verified current-data bug:
- `BUG-GPET-001` — tracked egg 55413 creates upbringing VNUM 55713, but `PET_HATCH_INFO_RANGE` has only 12 rows. Both ordinary hatching and enabled PET_ATTR_DETERMINE derive index `55713 - 55701 = 12` and read out of bounds.

Deferred test:
- `GPET-T01` — isolated debug/ASan boundary validation, not run.

## Exact next work
1. map hatch packet/UI -> server -> DB -> seal creation atomically;
2. map summon/dismiss/death/revive and lifetime persistence;
3. map mob/item EXP arithmetic and pet_exp_table boundaries;
4. map evolution materials/count/consumption;
5. map feed windows and item-slot trust boundaries;
6. map skill learn/upgrade/delete/passive/auto-skill execution;
7. audit PET_ATTR_DETERMINE and name-change inputs;
8. close disconnect/warp/owner destruction and DB lifetime;
9. map current 55701..55713 proto/race/client visual coverage;
10. consolidate Growth Pet runtime ownership only after static closure.

Do not execute `GPET-T01` or any runtime test.
The global future runtime order remains locked with `DUNGEON-T10` first.

GitHub state is canonical.
