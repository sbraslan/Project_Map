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
