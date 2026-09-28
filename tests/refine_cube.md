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


## REFCUBE-T08 — Sessionless / remote normal refine
Covers `BUG-REFCUBE-008`.

Future isolated modified-client test:
1. use a disposable refinable inventory item with required materials/currency;
2. ensure no refine dialog is open and no blacksmith is nearby;
3. submit one `HEADER_CG_REFINE` request with `REFINE_TYPE_NORMAL`;
4. capture HackLog plus item/material/currency result.

Static prediction:
the missing blacksmith is logged but does not abort; the normal refine transaction proceeds if its economic/item checks pass.

Safety class: **Stage B isolated modified-client / disposable refinement**.


## REFCUBE-T09 — Soul Awake refine-type routing
Covers `BUG-REFCUBE-009`.

Future controlled disposable test:
1. prepare an inactive disposable ITEM_SOUL compatible with the tracked Soul scroll flow;
2. use VNUM 70603 (Soul Awake parchment) through the normal scroll-on-target UI path;
3. record the refine-information type shown/sent and the server dispatch after confirmation;
4. compare with VNUM 70602 (Soul Evolve parchment).

Static prediction:
- 70602 maps to `REFINE_TYPE_SOUL_EVOLVE` and reaches `DoRefineSoul()`;
- 70603 leaves the request at generic `REFINE_TYPE_SCROLL` because the second condition repeats the EVOLVE value, so confirmation is routed to `DoRefineWithScroll()` instead of the dedicated Soul Awake path.

Safety class: **Stage A/B controlled disposable Soul-item flow**.


## REFCUBE-T10 — Refine ability skill probability direction
Covers `BUG-REFCUBE-010`.

Future deterministic debug/statistical validation:
1. choose a disposable recipe with a known base probability;
2. compare normal refine at skill 0 and a positive refine-skill level;
3. separately compare guild/money-only if available;
4. instrument the random roll and displayed probability.

Static prediction:
the positive skill value is added to the random roll rather than the threshold, lowering the real success rate while the refine-information packet reports an increase.

Safety class: **Stage A controlled probability/debug validation**.

## REFCUBE-T11 — Scroll preview vs execution probability
Covers `BUG-REFCUBE-011`.

Future deterministic validation:
1. use disposable targets at known refine levels;
2. capture displayed `TPacketGCRefineInformation.prob` for Magic Stone, Dragon Scroll, Smith Handbook and BDragon;
3. instrument the `DoRefineWithScroll()` `success_prob` selected for the same attempts;
4. compare formula/result without relying on random outcomes.

Static prediction:
the preview adds refine skill and treats several absolute scroll probability tables as additive buffs, while execution uses different formulas.

Safety class: **Stage A debug/formula validation**.
