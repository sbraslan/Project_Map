# Classic Quest Dungeons — Static System Map

**Status:** STATIC MAPPING OPEN / 0 VERIFIED BUGS  
**Mode:** detection / mapping only  
**Execution:** LOCKED / NOT RUN  
**Source policy:** Project_ClientSrc, Project_ServerSRC, Project_Binary, Project_Game and Project_DumpProto are read-only.

## Canonical scope
This node owns the enabled, quest-driven dungeon family that runs on the generic Dungeon Core rather than a dedicated C++ dungeon manager:
- Devil Tower;
- Devil Catacombs;
- Spider Dungeon / Spider Baroness;
- Flame Dungeon / Razador;
- Snow Dungeon / Nemere.

The already-closed generic Dungeon Core is not rescanned. This audit only follows feature-specific quest/data rules and their calls into already-mapped generic dungeon APIs.

## Deployment proof
The pinned ServerSRC snapshot enables the Devil Tower, Devil Catacombs, Spider Dungeon, Flame Dungeon and Snow Dungeon feature flags.

The pinned Game snapshot's active quest_list includes each corresponding quest source. Their quest/object outputs and dungeon regen/map data are also present.

## Audit cursor
1. Map each dungeon entry authorization, party/item/level requirements and private-map creation.
2. Map quest flags/server timers and floor/stage progression.
3. Audit item/reward consumption, replay/duplicate behavior and failure rollback.
4. Audit disconnect/reconnect/party-change and timeout/exit cleanup.
5. Audit feature-specific calls into generic Dungeon Core for cross-system bug reachability.
6. Close deployment/data parity and promote only source-proven defects.

## Boundary
Blue Dragon/Beran and Meley/DragonLair are not in this node because dedicated C++ components exist for those families. Dawnmist/Snake/White Dragon/Defense Wave are also deferred to later candidate families.
