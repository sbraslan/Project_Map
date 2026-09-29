# Classic Quest Dungeons — Static System Map

**Status:** STATIC MAPPING CLOSED / 4 VERIFIED BUGS  
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


## Cursor 3 finding — Catacomb entry item rollback

Devil Catacombs consumes the rag/golden-lock entry item before the private dungeon exists:
1. `item.remove()`;
2. two dialog `wait()` suspensions;
3. `d.new_jump_party(...)`;
4. dungeon state/regen initialization.

The C++ `d.new_jump_party` binding can fail when `CDungeonManager::Create` / private-map creation fails, but it returns no Lua success value and the quest has no rollback path. The consumed entry item therefore cannot be restored by this flow.

Promoted as `BUG-CLD-003`.

Flame ticket handling was separately checked: initial party validation is rechecked on private-map login while `dungeon_enter == 0`; a missing ticket schedules removal from the dungeon rather than silently granting a valid paid entry.


## Cursor 3 finding — Snow stage timers use stale global server-timer context

Four Snow Dungeon transitions call `server_timer(..., get_server_timer_arg())` from handlers that are **not** server-timer callbacks:
- `LEVEL2_KEY.use` -> floor 3;
- `LEVEL8_KEY.use` -> floor 9;
- `LEVEL5_CUBE.take` -> floor 6;
- `LEVEL6_STONE.kill` -> floor 7.

The C++ quest manager sets `m_dwServerTimerArg` only in `CQuestManager::ServerTimer(npc,arg)`. The value is singleton manager state and is not reset for normal item/take/kill events. Therefore these handlers do not obtain the current Snow private-map index; they reuse whichever server-timer arg most recently ran on that game process.

Promoted as `BUG-CLD-004`.


## Cursor 3 checkpoint — item/reward consumption / rollback
- Devil Tower key/map drops and Snow key/cube progression were traced through their consume/success/failure branches.
- Snow's wrong-order cube/key consumption is explicit quest behavior and was not promoted as a defect without contrary deployment/design evidence.
- Flame entry tickets are validated before instance creation and revalidated on private-map login before the run is started; the mission NPC then consumes valid tickets from the in-map party and ejects a member lacking a ticket.
- Devil Catacombs' consume-before-create transaction produced `BUG-CLD-003`.
- Snow's misuse of stale `get_server_timer_arg()` in non-timer progression events produced `BUG-CLD-004`.

No additional source-proven item/reward duplicate or rollback defect was promoted.


## Cursor 4 checkpoint — reconnect / party / timeout cleanup
- Snow's stale leader-absence timer is already captured as `BUG-CLD-002`.
- Flame rejoin is bound to the stored party dungeon index, matching leader PID and a per-player 5-minute exit window.
- Devil Catacombs validates reconnecting private-map players against the dungeon floor / player quest-floor state and schedules removal for a mismatched return.
- Devil Tower resets the exit warp target and removes its transient tower key/map items on logout.
- Spider Baroness uses its shared-room timeout/dead timers to clear channel-scoped run state and purge/warp the room.
- No additional source-proven reconnect, party-change or timeout cleanup defect was promoted.

## Cursor 5 checkpoint — generic Dungeon Core / Party dependency reachability
This cursor was a narrow dependency lookup only; the CLOSED Dungeon Core and Party nodes were not reopened.

- Classic source quests do not establish live reachability to `BUG-DUNGEON-001`: the affected generic APIs are `d.join` / `d.new_jump_guild`, while the mapped classic entry flows use `d.new_jump_party` / `d.new_jump_all`.
- No classic path uses `d.spawn_move_unique`, so `BUG-DUNGEON-002` is not duplicated here.
- The inspected `d.set_unique` uses employ distinct/generated keys; no feature-specific path was established for `BUG-DUNGEON-003` or `BUG-DUNGEON-004`.
- Devil Catacombs does have a live cross-system path through `d.exit_all_by_item_group("reapers_credit")`. Generic Dungeon Core can call `CParty::Quit(pid)` for a party member without the required item; if that member is the leader in a party with more than two members, execution reaches the already verified `BUG-PARTY-001` leader self-delete/use-after-free path.
- This is recorded as **Classic Catacomb -> Dungeon Core -> BUG-PARTY-001 reachability**, not assigned a duplicate `BUG-CLD-*` ID.

No new Classic-specific bug was promoted in this dependency cursor.


## Final deployment/data parity closure
- ServerSRC enables `ENABLE_DEVIL_TOWER`, `ENABLE_DEVIL_CATACOMBS`, `ENABLE_SPIDER_DUNGEON`, `ENABLE_FLAME_DUNGEON` and `ENABLE_SNOW_DUNGEON`.
- The active Game `quest_list` deploys Devil Tower, Devil Catacombs, both Spider quests, Flame Dungeon and Snow Dungeon.
- Compiled `quest/object` handlers are present for the mapped entry NPCs/items, kill events, login/logout hooks and timer callbacks.
- Matching map and dungeon-data trees are present: Devil Tower regen/map data, Catacomb dc_1f..dc_7f data, Spider dungeon maps/objects, Flame fd_* data/maps, and Snow sd_1..sd_10 data/map.
- No additional source-proven deployment/data mismatch was found.

## Static closure
Classic Quest Dungeons is **STATIC MAPPING CLOSED** for the pinned source snapshot.

Verified feature-specific bugs:
- `BUG-CLD-001` — Spider Baroness unsuffixed global boss VID causes cross-channel identity collision.
- `BUG-CLD-002` — Snow leader valid rejoin does not cancel stale leader-out shutdown timer.
- `BUG-CLD-003` — Catacomb entry item is consumed before fallible private-map creation with no rollback.
- `BUG-CLD-004` — Snow non-timer progression events use stale singleton `get_server_timer_arg()` as the next-stage instance key.

Cross-system reachability retained without duplicate ID:
- Devil Catacomb -> Dungeon Core item-group exit -> `BUG-PARTY-001`.

Runtime execution remains locked.
