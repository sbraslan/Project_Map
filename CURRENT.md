# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Refine / Cube / Crafting Static Mapping  
**Status:** STATIC MAPPING IN PROGRESS / 2 VERIFIED STATIC BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Refine / Cube / Crafting  
**System:** `systems/refine_cube.md`  
**Bugs:** `bugs/refine_cube.md`  
**Tests:** `tests/refine_cube.md`  
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

Canonical deferred tests:
`AURA-T01..AURA-T06`, none executed.

Closure includes:
- post-open distance authorization bypass;
- EVOLVE split-stack material underpayment;
- Aura Eraser generic socket2 mutation;
- uninitialized Aura SET_ITEM metadata;
- Yohara random-apply loss on successful EVOLVE;
- terminal Radiant EVOLVE preview zero-denominator arithmetic;
- current proto/refine-chain, booster, visual, disconnect and resource-data closure.

No source repository was modified.

## Active Refine / Cube / Crafting state

Current live Cube path is Cube Renewal (`ENABLE_CUBE_RENEWAL`).

Verified static bugs:
- `BUG-REFCUBE-001` — signed client multiplier has no server lower/upper bound; negative multiplier can reverse Yang/Gem recipe charges into currency credits.
- `BUG-REFCUBE-002` — legitimate multiplier >1 is used for availability and currency cost, but removable materials and reward quantity use only base recipe counts.

Deferred tests:
- `REFCUBE-T01..REFCUBE-T02`, none executed.

## Exact next work
1. map Cube open/NPC/distance/window authorization;
2. audit improve-item chance accounting and lifetime;
3. audit allow_copy / set_value / not_remove behavior;
4. close Cube success/failure/inventory-space atomicity;
5. map classic refine request -> server execution;
6. audit refine scroll/blacksmith/guild refine and failure downgrade paths;
7. audit socket/attribute/Yohara/element/set metadata preservation;
8. validate current cube.txt and refine deployment data;
9. consolidate runtime ownership only after static mapping is complete.

Do not execute `REFCUBE-T01..REFCUBE-T02` or any other runtime test.
The global future runtime order remains locked with `DUNGEON-T10` first.

GitHub state is canonical.
