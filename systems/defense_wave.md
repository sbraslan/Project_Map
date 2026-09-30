# Defense Wave / Ship Defense — Integrated Feature Evidence

**Lifecycle:** ACTIVE_INTEGRATED_SUBFEATURE  
**Canonical node:** no  
**Source snapshot:** pinned by `MAP_STATE.json`

## Active source proof
- ServerSRC enables `ENABLE_DEFENSE_WAVE`.
- Character code recognizes the Defense Wave portal, mast, Hydra bosses and Defense Wave mobs.
- The dedicated portal participates in the generic warp-NPC event and transports players to the Defense Wave/new-continent port area.
- Generic `CDungeon` is extended with a mast pointer and mast-HP synchronization.
- Dungeon Lua APIs are extended with mast lookup/targeting and unique-master helpers.
- Combat/restart/item hooks contain live Defense Wave behavior.
- Defense Wave maps, wave regen data and monster assets are tracked.

## Ownership boundary
There is no dedicated Defense Wave manager or independent registry. Private-run state is stored in the existing generic `CDungeon` object:
- mast ownership;
- dungeon flags/unique entities;
- participant/private-map lifecycle;
- generic dungeon Lua state.

The pinned Game `quest_list` contains no Defense Wave quest. Port NPCs in the tracked map have no matching questnpc/object bindings that establish a separate Defense Wave entry lifecycle.

## Classification
Defense Wave is an **active integrated extension of Dungeon Core**, not a separate canonical lifecycle node on this source snapshot. Opening it as a standalone node would duplicate the already-CLOSED generic dungeon ownership model.

## Reclassify when
- a dedicated Defense Wave manager/registry appears;
- a tracked independently deployed entry/state quest is added and owns lifecycle beyond generic `CDungeon`;
- persistent/networked Defense Wave-owned state appears;
- or the relevant source snapshot changes.
