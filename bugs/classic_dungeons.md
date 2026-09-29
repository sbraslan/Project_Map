# Classic Quest Dungeons — Bug Registry

**Status:** STATIC MAPPING OPEN / 3 VERIFIED BUGS  
**Execution:** LOCKED / NOT RUN

Previously verified generic Dungeon Core or Party bugs are referenced rather than duplicated unless a distinct feature-specific defect is proven.

## BUG-CLD-001 — Spider Baroness boss VID is stored in a cross-channel global event flag

**Class:** multiplayer state isolation / cross-channel identifier collision

### Proof
- Spider Baroness intentionally scopes its run state by channel for `spider_lair_ongoing_<channel>`, leader/start/end state, key state and `remain_egg<channel>`.
- When item 30327 spawns boss 2092, the quest instead stores the returned VID with `game.set_event_flag("king_vid", kingVid)`, without a channel suffix.
- Egg-kill handling later reads that same unsuffixed `king_vid` and passes it to `npc.set_vid_attack_mul` and `npc.set_vid_damage_mul`.
- `game.set_event_flag` calls `CQuestManager::RequestSetEventFlag`, which sends the value to DB with `HEADER_GD_SET_EVENT_FLAG`.
- DB `CClientManager::SetEventFlag` broadcasts `HEADER_DG_SET_EVENT_FLAG` to game peers and persists the flag as a PID-0 quest/event flag. New game peers also receive the complete event-flag set during setup.
- The NPC multiplier functions resolve the numeric VID locally with `CHARACTER_MANAGER::Find(vid)`.

### Consequence
Two channels running Spider Baroness concurrently share one `king_vid` slot. The later boss spawn overwrites the identifier seen by the other channel. Egg kills can then fail to scale the intended boss; if the overwritten numeric VID resolves to another local entity, the multiplier can be applied to the wrong character.

### Deferred validation
`CLD-T01`.



## BUG-CLD-002 — Snow Dungeon leader reconnect does not cancel the leader-out shutdown timer

**Class:** reconnect lifecycle / stale server timer

### Proof
- Snow Dungeon defines `REJOIN_LIMIT_TIME = 5` minutes.
- On logout from a Snow private map, if the character is party leader, the quest starts `snow_dungeon_leader_out_timer` with the current private-map index.
- The entry NPC explicitly supports rejoining the existing dungeon when the stored dungeon index and leader PID match and the player's exit time is within `REJOIN_LIMIT_TIME`.
- The normal private-map `when login` path also accepts a returning player, but neither return path clears `snow_dungeon_leader_out_timer`.
- The only normal clear of that timer is inside the general `snow_dungeon.clear_timer(inx)` teardown helper.
- When the stale leader-out timer fires, it schedules `snow_dungeon_end_timer`; that handler calls `snow_dungeon.clear_timer` and `d.exit_all()`.

### Consequence
A leader can return within the advertised rejoin window and continue the dungeon, yet the original logout timer still expires and terminates the active instance for the entire party.

### Deferred validation
`CLD-T02`.


## BUG-CLD-003 — Devil Catacombs consumes the entry item before fallible private-map creation

**Class:** item transaction / failure rollback

### Proof
- The 30101 take handler validates the item and party state, then immediately calls `item.remove()`.
- The quest performs two `wait()` suspensions before calling `d.new_jump_party`.
- `d.new_jump_party` calls `CDungeonManager::Create(mapIndex)`.
- If private-map creation fails, the binding logs `cannot create dungeon` and returns to Lua without pushing a success/failure result.
- The quest does not test a result and contains no compensation path that restores the removed item.

### Consequence
A failed instance-creation attempt can consume the Catacomb entry item without creating/entering the dungeon. The transaction is not atomic from the player's inventory perspective.

### Deferred validation
`CLD-T03`.
