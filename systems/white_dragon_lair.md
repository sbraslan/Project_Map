# White Dragon / Alastor — Dormant Candidate Evidence

**Lifecycle:** DORMANT_NO_DEPLOY_CALLER  
**Canonical node:** no  
**Source snapshot:** pinned by `MAP_STATE.json`

## Source presence
- ServerSRC enables `ENABLE_WHITE_DRAGON`.
- Dedicated `WhiteDragon::CWhDr` / `CWhDrMap` implementation exists.
- Lua bindings expose `WhiteDragon.Access`, `WhiteDragon.Access2` and `WhiteDragon.StartDungeon`.
- White Dragon / Alastor map and monster assets are tracked.
- Combat and Sungma integration hooks recognize White Dragon private maps.

## Deployment-caller check
- The pinned Game `quest_list` contains no White Dragon / Alastor quest.
- No tracked White Dragon quest-source/object path is present.
- The inspected non-quest server integration points do not call `CWhDr::Access`, `Access2` or `StartDungeon`; entry creation is reachable only through the Lua binding surface in the tracked source.

## Classification
The feature is source-complete enough to retain as a candidate, but there is no tracked deployment caller proving that players can create/start an instance in this snapshot.

Do not open a canonical mapping node on this snapshot.

## Reopen when
- a tracked quest/object caller to `WhiteDragon.Access/Access2/StartDungeon` appears;
- a non-quest caller to `CWhDr::Access/StartDungeon` is proven;
- or the relevant source snapshot changes.
