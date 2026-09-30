# Elemental World — Integrated World-Zone Evidence

**Lifecycle:** ACTIVE_INTEGRATED_WORLD_ZONE  
**Canonical node:** no  
**Source snapshot:** pinned by `MAP_STATE.json`

## Active source proof
- Project_Game tracks `metin2_map_elemental_01` through `metin2_map_elemental_04`.
- The maps use ordinary `MapSetting`, `Town.txt`, regen/NPC files and standard world-map data.
- Elemental map NPC files expose ordinary warp NPCs (for example vnums 3949, 10109, 10110, 10112, 10116, 10138) rather than a dedicated Elemental World manager entrypoint.
- Project_Binary tracks the corresponding elemental map texture/atlas data.
- Shared elemental/Sungma combat and progression behavior belongs to already-mapped Yohara/Sungma infrastructure rather than a separate persistent Elemental World state machine.

## Ownership boundary
No dedicated Elemental World manager, registry, independent packet family, player-table persistence block, or private-run lifecycle was proven on the pinned source snapshot.

The feature family is therefore a set of active world zones wired through generic map/warp infrastructure and shared Yohara/Sungma mechanics.

## Classification
Elemental World is an **active integrated world-zone family**, not a separate canonical lifecycle node.

Opening it as a standalone subsystem would duplicate:
- generic map/warp ownership; and
- the already-CLOSED Yohara progression/Sungma ownership model.

## Reclassify when
- a dedicated Elemental World manager/registry appears;
- independent persistent or networked Elemental World-owned state appears;
- a dedicated deployed quest/service owns entry/session lifecycle beyond generic warp/map behavior;
- or the relevant source snapshot changes.
