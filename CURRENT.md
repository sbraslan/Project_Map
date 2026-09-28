# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Refine / Cube / Crafting Static Mapping  
**Status:** STATIC MAPPING IN PROGRESS / 11 VERIFIED STATIC BUGS / EXECUTION LOCKED  
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
- `BUG-REFCUBE-001` — Cube multiplier domain is not validated; zero/negative values break resource/currency semantics.
- `BUG-REFCUBE-002` — positive batch multiplier is omitted from actual removable-material consumption and reward quantity.
- `BUG-REFCUBE-003` — Cube MAKE remains authorized after close/range through stale cached NPC VNUM.
- `BUG-REFCUBE-004` — player-level `/cube` can dereference a null quest NPC.
- `BUG-REFCUBE-005` — Cube chance-improve item can be consumed before reward-space abort.
- `BUG-REFCUBE-006` — current Cube recipes read uninitialized `allow_copy` / missing `not_remove` control fields.
- `BUG-REFCUBE-007` — classic refine replacement paths dereference the destroyed source item after `RemoveItem()`.
- `BUG-REFCUBE-008` — `REFINE_TYPE_NORMAL` is not bound to an active refine session or enforced blacksmith proximity.
- `BUG-REFCUBE-009` — Soul Awake scroll is misrouted as generic scroll because the type comparison repeats EVOLVE.
- `BUG-REFCUBE-010` — positive refine-skill bonuses are added to the RNG roll, lowering actual success while UI reports an increase.
- `BUG-REFCUBE-011` — scroll refine probability shown by `RefineInformation()` diverges from the formula used by `DoRefineWithScroll()`.

Deferred tests:
- `REFCUBE-T01..REFCUBE-T11`, none executed.

Static closure already covers:
- Cube packet/accounting/open-state/improve-item/control-field boundaries;
- current 3327-section `cube.txt` control-directive reachability;
- classic `HEADER_CG_REFINE` dispatch;
- normal/scroll/Serpent source lifetime;
- Soul scroll routing;
- classic metadata-copy matrix;
- all 5587 tracked `RefinedVnum` topology edges for missing target / size / type / subtype changes;
- refine-skill and scroll preview-vs-execution probability formulas.

## Exact next work
1. map DB/refine-table deployment source and validate recipe probability/material bounds;
2. audit random-default / set / transmutation metadata semantics only where current refine data makes them reachable;
3. close scroll failure/downgrade consumption and item-creation-failure ordering;
4. audit Devil Tower / money-only / Serpent authorization lifetime;
5. audit over-9 refine path and current deployment reachability;
6. close client `uirefine.py` session/presentation lifecycle;
7. consolidate runtime ownership and decide STATIC COMPLETE.

Do not execute `REFCUBE-T01..REFCUBE-T11` or any other runtime test.
The global future runtime order remains locked with `DUNGEON-T10` first.

GitHub state is canonical.

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

Do not execute `REFCUBE-T01..REFCUBE-T05` or any other runtime test.
The global future runtime order remains locked with `DUNGEON-T10` first.

GitHub state is canonical.
