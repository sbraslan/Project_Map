# Refine / Cube / Crafting — Deferred Tests

**Execution status:** LOCKED / NOT RUN

## REFCUBE-T01 — Non-positive Cube Renewal multiplier
Covers `BUG-REFCUBE-001`.

Future isolated modified-client test:
1. use a disposable recipe with a nonzero Yang and/or Gem cost;
2. record currency and required materials;
3. in separate isolated cases submit the normal recipe VNUM/material list with multiplier 0 and with a negative multiplier;
4. record material, reward and currency deltas;
5. stop after one request.

Static prediction:
- zero multiplier collapses multiplied requirements/costs to zero and can still reach reward creation;
- negative multiplier passes negative requirement products and reverses currency mutation into a credit.

Safety class: **Stage B isolated modified-client / economy mutation**.
Never run against production economy.

## REFCUBE-T02 — Normal Cube Renewal multiplier > 1 accounting
Covers `BUG-REFCUBE-002`.

Future controlled disposable-data test:
1. choose a stackable-output recipe available in current `cube.txt`;
2. prepare enough materials/currency for multiplier 2;
3. use the normal UI to request multiplier 2;
4. record before/after material, Yang/Gem and reward quantities.

Static prediction:
availability/currency follow multiplier 2, but removable materials and created reward use only the base recipe count.

Safety class: **Stage A/B controlled disposable recipe**.

Global execution remains locked; `DUNGEON-T10` is still the first future live gate.


## REFCUBE-T03 — Craft after Cube close / out of NPC range
Covers `BUG-REFCUBE-003`.

Future isolated modified-client test:
1. legitimately open a Cube Renewal NPC;
2. close the Cube window and/or move clearly outside NPC interaction range;
3. retain a disposable valid recipe for that NPC;
4. submit one MAKE packet without reopening Cube;
5. record whether the recipe executes.

Static prediction:
the server uses stale `tempCubeNPC` and does not require open state or distance.

Safety class: **Stage B isolated authorization / disposable recipe**.

## REFCUBE-T04 — Direct /cube without quest NPC
Covers `BUG-REFCUBE-004`.

Future isolated debug/sanitizer test:
1. use a test character with no current quest NPC;
2. invoke the player-level `/cube` command once;
3. observe the `GetQuestNPC()->GetRaceNum()` path under debugger/sanitizer.

Static prediction:
null pointer dereference before a Cube open packet is built.

Safety class: **Stage C crash/sanitizer only**.
Never run on production.

## REFCUBE-T05 — Improve-item loss on no reward space
Covers `BUG-REFCUBE-005`.

Future controlled disposable-data test:
1. prepare a valid <100% Cube recipe;
2. attach disposable VNUM 79605 improve items;
3. ensure no valid reward destination slot exists;
4. submit one normal craft attempt;
5. compare improve-item, recipe-material and currency deltas.

Static prediction:
improve items decrease first; reward-space check then aborts; base materials/currency remain unchanged.

Safety class: **Stage A controlled disposable-item data-integrity test**.


## REFCUBE-T06 — Uninitialized Cube recipe controls
Covers `BUG-REFCUBE-006`.

Future isolated debug validation:
1. load the current `cube.txt` under a debug/MemorySanitizer build;
2. inspect representative recipes with no `allow_copy` directive and recipes with no `not_remove` directive;
3. observe the parsed `CUBE_DATA` values before `RefineCube()`;
4. compare behavior against an explicitly zero-initialized control build.

Static prediction:
the omitted scalar fields are read without initialization; current recipes can enter copy/not-remove branches nondeterministically.

Safety class: **Stage C debug / sanitizer only**.


## REFCUBE-T07 — Classic refine source lifetime after successful replacement
Covers `BUG-REFCUBE-007`.

Future isolated ASan/debug test:
1. use a disposable ordinary non-Metin refinable item;
2. execute one successful normal refine with Battle Pass code active;
3. separately cover scroll success and, where reachable, scroll grade-down / Serpent paths;
4. instrument `ITEM_MANAGER::RemoveItem()` and all subsequent old-source accesses.

Static prediction:
`RemoveItem()` destroys the source CItem before later announcement/Yohara/Battle-Pass code dereferences the stale pointer.

Safety class: **Stage C crash/lifetime/sanitizer only**.
