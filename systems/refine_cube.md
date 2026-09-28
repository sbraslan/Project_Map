# Refine / Cube / Crafting — Static Map

**Status:** STATIC MAPPING IN PROGRESS  
**Phase:** Detection / Mapping Only  
**Source policy:** source/game repositories read-only; only Project_Map may be edited.

## Scope opened — 2026-09-28

Server surfaces:
- `game/src/refine.cpp` / `refine.h` — refine recipe manager;
- `game/src/char_item.cpp` — item refine execution paths;
- `game/src/over9refine.cpp` — over-9 refinement;
- `game/src/CubeManager.cpp` / `CubeManager.h` — active Cube Renewal path;
- `game/src/cube.cpp` / `cube.h` — legacy cube path, compiled out while ENABLE_CUBE_RENEWAL is active;
- `game/src/input_main.cpp::CubeRenewalSend()` — Cube Renewal client packet dispatch.

Client surfaces:
- `UserInterface/PythonCubeRenewal.cpp`;
- `UserInterface/PythonNetworkStreamPhaseGame.cpp::SendCubeRefinePacket()`;
- `root/uicuberenewal.py`;
- `root/uirefine.py`.

Deployment/config:
- locale `cube.txt`;
- DB/server refine table loaded through `CRefineManager`;
- item proto refine-set/refined-vnum relationships.

Current build has `ENABLE_CUBE_RENEWAL` enabled, so `CubeManager.cpp` is the primary live Cube implementation and the legacy `cube.cpp` path is conditional/dormant.

## Cube Renewal packet contract

Client packet:
`TSubPacketCGCubeRenwalMake`
contains:
- `int vnum`;
- `int multiplier`;
- `int indexImprove`;
- `int itemReq[5]`.

Normal UI starts multiplier at 1 and permits stackable-result recipes to increase it up to 200.

Server `CInputMain::CubeRenewalSend()` accepts the packet multiplier directly and passes it to:
`CCubeManager::RefineCube(ch, vnum, multiplier, indexImprove, itemReq)`.

No server-side lower or upper bound is applied before economic arithmetic.

## First verified findings

### Multiplier domain is not validated
For materials, Yang and Gem preconditions the server multiplies recipe requirements by the client-supplied `multiplier`.

A negative multiplier makes positive inventory/currency values compare against negative requirements, so those gates pass.

Later currency mutation uses:
- `PointChange(POINT_GOLD, -(gold * multiplier), ...)`;
- `PointChange(POINT_GEM, -(gem_point * multiplier), ...)`.

With a negative multiplier the sign reverses and the craft can credit currency rather than charge it.

See `BUG-REFCUBE-001`.

### Positive multiplier is not applied consistently
For a legitimate multiplier > 1:
- material availability is checked against `material.count * multiplier`;
- Yang/Gem are charged with `* multiplier`;
- UI displays material/reward/currency quantities with `* multiplier`.

But server material removal uses only:
`RemoveSpecifyItem(Material.vnum, Material.count, ...)`

and normal reward creation uses only:
`CreateItem(itemVnum, itemCount)`.

Thus one multi-craft request validates as a batch and charges batch currency but consumes only one recipe's removable materials and creates only one recipe's reward count.

See `BUG-REFCUBE-002`.

## Immediate next audit
1. map Cube open/NPC/distance/window authorization;
2. audit `indexImprove` chance-item accounting and lifetime;
3. audit `allow_copy`, `set_value`, `not_remove` ownership and item-copy semantics;
4. map Cube success/failure atomicity and inventory-space ordering;
5. map classic refine request -> server refine execution;
6. audit refine scroll, guild/blacksmith, Yang/material and failure-downgrade paths;
7. audit socket/attribute/Yohara/element/set metadata preservation across refine;
8. validate current `cube.txt` and refine-table deployment data.

No runtime test is authorized. Global first future live gate remains `DUNGEON-T10`.


## Cube authorization / improve-item pass — 2026-09-28

### Cached NPC authorization
Renewal `/cube` open stores both:
- a live NPC pointer through `SetCubeNpc(GetQuestNPC())`;
- the NPC race through `SetTempCubeNPC(GetQuestNPC()->GetRaceNum())`.

Close clears the live pointer / `W_CUBE` but leaves `tempCubeNPC` unchanged.

MAKE does not require open state, live NPC, `W_CUBE`, or NPC distance. Recipe selection is driven by the cached temporary NPC VNUM.

This is `BUG-REFCUBE-003`.

### Player command null boundary
The command table exposes `cube` at `GM_PLAYER`.

A character initializes `m_dwQuestNPCVID=0`; `GetQuestNPC()` resolves that VID and can return null.

Renewal `do_cube()` has no null check before `GetQuestNPC()->GetRaceNum()`.

This is `BUG-REFCUBE-004`.

### Improve item ordering
VNUM 79605 is consumed immediately after its count is converted into added success chance.

Reward inventory-space validation happens afterward using a temporary reward item.

Therefore a no-space abort can occur after the chance item has already been consumed.

This is `BUG-REFCUBE-005`.

### Current cube.txt reachability
Current deployment file has about 3.3k parsed sections and actively uses:
- `not_remove` extensively;
- `set_value` extensively;
- percentage values from 2..100;
- Yang and, for set recipes, Gem costs.

No current `allow_copy=1` section was found, so that branch is mapped but presently deployment-dormant.

Set-value recipes are real/current, e.g. NPC 20475 recipes preserve the source item with `not_remove`, charge 100,000,000 Yang + 500 Gem, and set set-value 1 on success.

## Current verified findings
`BUG-REFCUBE-001..BUG-REFCUBE-005`.

## Next
Continue into classic refine execution, scroll/failure paths and metadata preservation while finishing Cube special-branch atomicity.


## Classic refine / recipe-control pass — 2026-09-28

### Cube recipe object initialization
`CUBE_DATA` defines a user constructor that initializes only:
- `set_value = 0`;
- `gem_point = 0`.

At section start the parser also explicitly assigns `gold = 0`.

Other scalar control members are written only when their directive appears.

Current tracked `cube.txt` has **3327 complete sections**:
- `percent`: 3327 / 3327;
- `allow_copy`: 0 / 3327;
- `not_remove`: 2815 / 3327;
- `set_value`: 2790 / 3327.

Therefore:
- `allow_copy` is uninitialized for every current recipe;
- `not_remove` is uninitialized for 512 current recipes.

Both are read by `RefineCube()` to decide material removal / source preservation / attribute-copy behavior.

See `BUG-REFCUBE-006`.

### Current set-value recipe contract
All 2790 current `set_value` sections use:
- first material VNUM == reward VNUM;
- `not_remove` == that reward/source VNUM.

The stock Cube UI sends recipe/reward VNUM plus material **VNUMs**, not a chosen physical item cell. Therefore duplicate same-VNUM instance selection is not represented in the protocol; no separate “wrong selected instance” bug was promoted from this pass.

### Classic refine packet entry
Primary route:
`HEADER_CG_REFINE`
-> `CInputMain::Refine()`
-> one of:
- `DoRefine()`;
- `DoRefineWithScroll()`;
- `DoRefineSoul()`;
- `DoRefineSerpent()` through money-only SnakeLair branch.

`RefineInformation()` is the normal UI/info producer, but `CInputMain::Refine()` does not require a matching active refine session before honoring `REFINE_TYPE_NORMAL`.

`DoRefine()` calls `CanHandleItem(true)`, which intentionally bypasses the under-refine guard.

Its nearby-blacksmith scan logs `REFINE_FAR_BLACKSMITH` when none is present but deliberately continues.

See `BUG-REFCUBE-008`.

### Refine source lifetime
`ITEM_MANAGER::RemoveItem()` ends in `M2_DELETE(item)`.

Normal/scroll/Serpent refine replacement paths retain and use the old raw item pointer after that destruction.

Current enabled features make the normal-success path directly reachable:
- refine-success announcement;
- Yohara;
- Battle Pass.

The Battle Pass update alone calls `item->GetVnum()` after ordinary source destruction.

See `BUG-REFCUBE-007`.

### Metadata copy matrix
`ITEM_MANAGER::CopyAllAttrTo(old,new)` preserves:
- accessory sockets as-is;
- normal metin/socket state through its reconstruction rules;
- Yohara glove extra socket range;
- element grade/attacks/type/values;
- classic attributes;
- Yohara random apply attributes.

Refine callers additionally copy seal date.

It does **not** generically copy:
- item set value;
- Yohara random-default values;
- transmutation;
- basic-item marker.

Current set-value recipes provide explicit re-application routes for set-equipped VNUMs; no semantic bug is promoted yet from set-value reset alone.

Random-default preservation remains a data/semantics candidate pending current producer/consumer closure.

### Refine output proto topology
Current tracked item proto contains **5587** valid `RefinedVnum != 0` edges and no missing target VNUMs.

Across all edges:
- one size-changing edge exists;
- three subtype-changing edges exist;
- no type-changing edge exists.

All size/subtype anomalies have `RefineSet = 0`, so normal classic refine cannot obtain a recipe for them.

No currently executable refine edge was found that grows item size while the result is placed back into the old inventory cell.

### Soul refine path
Current build enables `ENABLE_SOUL_SYSTEM`.

Tracked Soul data includes:
- ITEM_SOUL families 70500..70509;
- evolve scroll 70602, Value0=8;
- awake scroll 70603, Value0=9.

`RefineItem()` correctly enters the shared Soul-scroll case, but its second condition repeats `SOUL_EVOLVE_SCROLL` rather than checking `SOUL_AWAKE_SCROLL`.

Thus Awake leaves `refType` at generic SCROLL and is later dispatched to `DoRefineWithScroll()`, not `DoRefineSoul()`.

See `BUG-REFCUBE-009`.

Soul socket lifecycle itself is currently coherent with replacement semantics:
- socket0 = real-time expiry;
- socket1 = activated marker (active Soul cannot be refined);
- socket2 is initialized from the new grade's Value2 and updated by Soul time-use logic;
- socket3 tracks Soul play time.

No additional Soul metadata-loss bug was promoted from this pass.

### Refine ability skill inversion
Current build enables `ENABLE_REFINE_ABILITY_SKILL`.

Positive tables:
- normal refine skill: 0..6;
- guild refine skill: 0..3.

`RefineInformation()` reports these as additions to the success percentage.

`DoRefine()` instead adds them to the random roll and compares that larger roll against the unchanged recipe threshold.

Therefore the skill reduces real success while UI reports an increase. Guild/money-only additionally adds 10 to the roll.

See `BUG-REFCUBE-010`.

### Scroll probability preview vs transaction
`RefineInformation()` builds non-guild scroll preview as:
`base recipe prob + refine skill + scroll_buff`.

`DoRefineWithScroll()` does not use refine skill and several scroll values are absolute probability tables rather than additive bonuses.

Affected examples include:
- Magic Stone;
- Dragon Scroll;
- Smith Handbook;
- BDragon;
- Ritual Stone;
- Seal of God.

The preview can therefore materially overstate the actual server transaction probability, including displaying 100% for attempts that execute below 100%.

See `BUG-REFCUBE-011`.

## Current verified findings
`BUG-REFCUBE-001..011`.

## Deferred runtime ownership
`REFCUBE-T01..REFCUBE-T11`.

No Refine/Cube runtime test has been executed.

## Remaining static work
1. map the DB/refine-table deployment source and validate recipe probability/material bounds;
2. audit current refine edges for random-default / set / transmutation semantics where those fields are actually produced;
3. close scroll failure/downgrade consumption and item-creation-failure ordering;
4. audit money-only / Devil Tower / Serpent authorization lifetime;
5. audit over-9 refine conditional path and current deployment reachability;
6. close client `uirefine.py` presentation/session lifecycle;
7. consolidate runtime ownership and decide STATIC COMPLETE.


## Closure pass — recipe source, metadata and dormant Over9 — 2026-09-28

### Refine recipe deployment source
Classic refine recipes are not loaded from a tracked text file by the game process.

DB boot executes a SELECT from `refine_proto` for id, cost, probability and up to five material VNUM/count pairs, then sends `TRefineTable` rows to game.

The DB loader zero-initializes each row and stops material parsing at the first zero VNUM, but it does not enforce semantic domains for probability, cost, material counts or duplicate IDs.

No SQL dump/current `refine_proto` dataset is tracked in the mapped repositories, so current production recipe values cannot be statically audited from GitHub. This is a deployment-data visibility limitation, not a promoted defect without a bad reachable row.

### Current Cube data domain audit
Tracked `share/locale/europe/cube.txt` contains 3327 complete sections.

Current observed domains:
- percent: 2..100;
- gold: 0..500,000,000;
- gem: 0..25,000;
- material VNUM/count: all positive in parsed sections;
- exactly one positive reward entry per parsed section.

No additional bad current Cube value was promoted from those domains.

The independent constructor/default problem remains `BUG-REFCUBE-006`: `allow_copy` is absent from every current section and `not_remove` is absent from 512 sections while the corresponding fields are not initialized by `CUBE_DATA()`.

### MONEY_ONLY / Devil Tower / Serpent authorization
`REFINE_TYPE_MONEY_ONLY` rechecks authorization at packet execution time.

Serpent:
- must currently be in a Snake map;
- must have passed the 24h `snake_lair.refine_time` gate.

Devil Tower:
- requires positive `deviltower_zone.can_refine`;
- decrements it only when `DoRefine(..., true)` reports success.

No second `BUG-REFCUBE-008`-style remote authorization issue was promoted for MONEY_ONLY.

### Scroll item-creation failure ordering
Scroll/material costs are consumed before the result `CreateItem()` call.

If result item creation unexpectedly fails, the source item can survive while consumed materials/scroll are not rolled back. This remains a robustness/fault-injection candidate because no tracked normal-data producer of result-item creation failure was established.

### Over-9 refine
`COver9RefineManager` is constructed and Lua bindings exist:
- `item.can_over9refine`;
- `item.change_to_over9`;
- `item.over9refine`;
- `item.get_over9_material_vnum`.

The tracked Project_Game quest source contains no current usage of those bindings, and no tracked startup/config path populating `m_mapItem` through `enableOver9Refine()` was found.

Its transform functions copy only sockets and normal attributes and would lose modern metadata, but without a tracked producer/active quest route this remains dormant and is not promoted.

### Refine client lifecycle
Normal client cancel/Escape sends `type=255`; server handles that value by `ClearRefineMode()`.

The significant current client presentation defect is instead `BUG-REFCUBE-012`: `RefineDialogNew.Open()` calls the set-value binding with the wrong two-argument signature.

### Transform metadata matrix
`ITEM_MANAGER::CopyAllAttrTo()` preserves:
- classic/accessory sockets;
- element state;
- normal item attributes;
- Yohara random apply attributes.

It does not preserve:
- ChangeLook/transmutation VNUM -> `BUG-REFCUBE-013`;
- Set Item `set_value` -> `BUG-REFCUBE-014`;
- Yohara/Serpent random-default values -> `BUG-REFCUBE-015`;
- Basic starter-item flag -> `BUG-REFCUBE-016`.

Seal date is copied explicitly by the mapped classic refine callers.

Tracked data confirms normal reachability:
- Set Smith recipes include +7/+8/+9 weapon families;
- Serpent ranges contain continuous +0..+15 families and generated random defaults affect combat;
- Basic starter equipment includes refinable body armor while `BLOCK_REFINE_ON_BASIC` is disabled;
- ChangeLook explicitly accepts ordinary weapons and body armor, and classic refine has no ChangeLook exclusion.

## Current verified findings
`BUG-REFCUBE-001..016`.

## Closure assessment
All mapped classic refine, scroll, money-only/Serpent, Cube Renewal, Over9 exposure, client refine UI and persistent transform-metadata boundaries have static ownership.

Remaining non-promoted boundaries are deployment/fault-injection dependent:
- untracked live DB `refine_proto` semantic values;
- result `CreateItem()` failure after pre-consumption;
- dormant Over9 metadata loss without a tracked producer.

No runtime test has been executed.
