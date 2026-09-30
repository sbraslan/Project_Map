# DawnMist / Temple of Ochao / Jotun — Integrated Feature Evidence

**Lifecycle:** ACTIVE_INTEGRATED_SUBFEATURE  
**Canonical node:** no  
**Source snapshot:** pinned by `MAP_STATE.json`

## Active source proof
- ServerSRC enables `ENABLE_DAWNMIST_DUNGEON`.
- `CDawnMistDungeon` is directly called from live combat paths:
  - Temple boss death spawns the Temple Guardian;
  - Guardian combat tracks fight/last-hit time;
  - Guardian death spawns the temporary portal;
  - Jotun/forest boss death and healer death paths are integrated;
  - Jotun healer waves and class-sensitive boss skill modifiers are live.
- Character death/restart logic recognizes DawnMist private-instance map indexes.
- Guard Compass item behavior is enabled for the Temple map.
- DawnMist/Ochao/Jotun maps and monster assets are tracked.

## Ownership boundary
`CDawnMistDungeon` does not own:
- private-map creation/registration;
- an entry/member registry;
- a party/dungeon object;
- independent player persistence;
- a dedicated network protocol.

It is a combat/map-mechanics layer attached to existing maps and generic/private-dungeon ownership elsewhere.

The pinned Game `quest_list` has no DawnMist/Ochao/Jotun quest entry. No independent deployable quest lifecycle is proven from the tracked snapshot.

## Classification
Keep DawnMist as an **active integrated subfeature**, not a standalone canonical lifecycle node. Its hooks are real and active, but opening a separate subsystem would duplicate map/dungeon ownership already represented elsewhere rather than identify an independently stateful system.

## Reclassify when
- a dedicated entry/private-map/member lifecycle is added;
- a persistent or networked DawnMist-owned state surface appears;
- or the relevant source snapshot changes.
