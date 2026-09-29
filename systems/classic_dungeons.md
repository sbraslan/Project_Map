# Classic Quest Dungeons — Static System Map

**Status:** STATIC MAPPING OPEN / 2 VERIFIED BUGS  
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


## Cursor 1 checkpoint — entry authorization / instance ownership

### Scope result
- Devil Tower enters through the public tower map and later creates a private map with `d.new_jump_all`.
- Devil Catacombs gates floor-1 entry by level, tower completion and cooldown; the rag transition requires a party and creates a private party dungeon.
- Spider 2F is a direct public-map warp.
- Spider Baroness is a channel-scoped shared-room design rather than a generic private dungeon.
- Flame and Snow validate the same-map party population and create private instances with `d.new_jump_party`.
- The Flame/Snow party-validation scope matches the server implementation of `d.new_jump_party`: both use `ForEachOnMapMember(..., sourceMapIndex)`. Off-map party members are therefore neither validated nor warped by that entry action; the initially suspected remote-member bypass is not a defect.

### Verified multiplayer defect
Spider Baroness correctly suffixes most shared state with `get_channel_id()`, but stores its boss VID in the unsuffixed event flag `king_vid`. Event flags are DB-owned/global: Game sends `HEADER_GD_SET_EVENT_FLAG`, DB persists the flag under PID 0 and broadcasts `HEADER_DG_SET_EVENT_FLAG` to game peers. A character VID, by contrast, is resolved locally through `CHARACTER_MANAGER::Find(vid)`.

Concurrent Spider Baroness runs on different channels can therefore overwrite the single global `king_vid`. Subsequent egg kills may fail to modify the intended boss or can target a different local character if that numeric VID exists on the receiving game process.

Promoted as `BUG-CLD-001`.

### Next cursor
Quest flags, server timers and floor/stage progression for the five dungeon families.


## Cursor 2 finding — Snow Dungeon leader reconnect timer

Snow Dungeon starts `snow_dungeon_leader_out_timer` for the private-map index whenever the party leader logs out. The configured `REJOIN_LIMIT_TIME` is 5 minutes.

The same quest explicitly permits rejoin within that 5-minute window, but neither:
- the private-map `when login` path, nor
- the ENTRY_MAN rejoin path

clears `snow_dungeon_leader_out_timer`.

When the original timer expires it unconditionally schedules `snow_dungeon_end_timer`, which then clears the instance timers and calls `d.exit_all()`. A leader can therefore successfully reconnect/rejoin and still have the active run forcibly terminated by the stale absence timer.

Promoted as `BUG-CLD-002`.


## Cursor 2 checkpoint — timers / stage progression
- Devil Tower, Devil Catacombs, Flame and Snow private-map server-timer handlers select their instance with `d.select(get_server_timer_arg())` before dungeon state mutation.
- Stage timers are keyed by the private map index and generic Dungeon teardown also cancels timers for the destroyed map index.
- Spider Baroness uses process-local server timers for its single shared room while channel-suffixed event flags hold run state; the already-promoted unsuffixed `king_vid` remains the isolation exception.
- Devil Catacombs contains `clear_server_timer("devilcatacomb_floor7_timer", 3, get_server_timer_arg())`; the Lua binding reads only name + second argument, so this call targets key 3. In the mapped normal flow the floor-7 transition timer has already fired before the exit handler reaches this line, so no independent runtime consequence is source-proven and it is not promoted.
- Snow leader absence/rejoin timer behavior produced `BUG-CLD-002`.

No additional timer/stage defect was promoted in this cursor.
