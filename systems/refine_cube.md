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
