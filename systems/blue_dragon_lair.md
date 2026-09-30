# Blue Dragon / Beran Setaou — Static System Map

**Status:** STATIC MAPPING CLOSED / 4 VERIFIED BUGS
**Mode:** detection / mapping only
**Execution:** LOCKED / NOT RUN

## Scope
Active Blue Dragon / Beran Setaou renewal flow: quest entry/state, map 208 ownership, global event state, access-item transaction, BlueDragon combat hooks, skill factors, lair regen/data and compiled quest handlers.

Meley / Red Dragon Lair is a separate canonical family because it has its own manager, participant, reward and ranking lifecycle.

The older `CDragonLairManager` / `DragonLair.startRaid` surface is dependency-only until a tracked deployed caller is proven.

## Audit cursor
1. Entry authority, global event state, channel serialization, group-entry window and access-item transaction.
2. Boss spawn/combat hooks, skill timers and stone modifiers.
3. Timeout, death, rejoin/login and room purge/warp lifecycle.
4. Legacy `DragonLair.startRaid` reachability.
5. Deployment/data parity and static closure.


## Cursor 1 finding — access-item loss on disconnect before delayed entry
The first entrant gives the required access items before the run is committed. The quest removes the items and then schedules the player timer `dragon_lair_warptimer` for `pc.get_channel_id() * 2` seconds.

Player quest timers are owned by the quest `PC` object. On character disconnect, `CQuestManager::DisconnectPC` erases that object; `PC::~PC -> Destroy -> ClearTimer` cancels its timers. If disconnect occurs after item removal but before `dragon_lair_warptimer` executes, neither the successful warp path nor the race-refund path runs.

Promoted as `BUG-BDL-001`.


## Cursor 1 finding — disconnect can strand the entry NPC lock
The first-entry path uses `npc.lock()` before several interactive suspend points. Normal quest completion eventually reaches `PC::EndRunning()`, which detects a locked quest NPC and clears `npc->SetQuestNPCID(0)`.

Disconnect follows a different path:
- `LogoutPC()` calls `CloseState()` and then `CancelRunning()`;
- `CloseState()` only unreferences the Lua coroutine;
- `CancelRunning()` only clears running-quest state;
- `PC::EndRunning()` is not called, so its NPC-unlock logic never executes.

The NPC therefore retains the disconnected player's PID in `GetQuestNPCID()`. Future `npc.lock()` calls reject other players because they only accept lock-owner 0 or the same PID.

Promoted as `BUG-BDL-002`.


## Cursor 1 finding — paid join remains open after the dragon is already dead
The existing-run entry branch is selected solely by `starttime + group_time >= current_time`. It does not require `dragon_lair_alive == 1`.

The boss-kill handler explicitly sets `dragon_lair_alive = 0` and purges the room, but leaves the original start time intact. If the boss dies during the first half of the cooldown window, new players can still pass the group-window branch, pay the access items and warp into the already-completed lair.

Promoted as `BUG-BDL-003`.

## Cursor 1 checkpoint — entry authority / deployment ownership
- Tracked entry NPC 30121 is spawned on map 73 and inside map 208.
- Map 73 and map 208 are both hosted on `ch1/core4`; the process-local Blue Dragon server timer therefore stays on the same core as the active lair in the tracked deployment.
- Global event flags serialize start time / room state across peers; the tracked deployment exposes only one active map-208 host.
- Entry transaction produced `BUG-BDL-001`, NPC lock lifecycle produced `BUG-BDL-002`, and dead-run paid join produced `BUG-BDL-003`.


## Cursor 2 finding — low-HP Blue Dragon factor ranges are inverted
`BlueDragon_GetRangeFactor(key, hpPct)` matches a row only when `min <= hpPct <= max`.

In `BlueDragon.lua`:
- `hp_damage[4].min = 30`, `max = 0`, expected pct 20;
- `hp_regen[4].min = 30`, `max = 0`, expected pct 12.

Those conditions can never be true. The damage factor is consumed by the active Blue Dragon skill functors, and the regen factor is consumed by the 2493 monster recovery event.

Promoted as `BUG-BDL-004`.

## Cursor 2 checkpoint — active vs dormant combat layers
- Group 2430 is not a boss-vnum mismatch: it is a group whose leader is 2493 Beran-Setaou.
- 2493 always receives `BlueDragon_StateBattle()`, so skill timing / BlueDragon.lua skill values are live on static map 208.
- The `ENABLE_BLUEDRAGON_RENEWAL` stone/damage guard in `char_battle.cpp` requires a private map index in the 208xxxx range plus a generic Dungeon pointer. The tracked quest uses static map 208 via `pc.warp()`, so that private-map-only branch is not reachable from this deployed quest flow.
- The tracked static lair data uses its older group/stone layout; the separate private-map renewal combat surface is therefore retained as dependency/dormant evidence until a deployed caller is found.

No other source-proven active combat defect was promoted in this cursor.


## Cursor 3 checkpoint — timeout / death / rejoin / room cleanup
- Boss-kill cleanup drops items before `purge_area`; the C++ purge helper destroys only monster/stone character entities, not item entities, so reward drops are preserved.
- The global timeout timer runs on the same tracked core that hosts maps 73 and 208.
- Personal countdown timers are reconnect-safe at quest level: disconnect cancels the personal timer, but `kill_dragon.login` recomputes remaining time and recreates it when the same run is still valid.
- Expired or mismatched returning characters transition back to `start`; the start state's enter/login path removes non-GM players from map 208.
- A timed-out living room is cleaned before the next leader run by `purge_area` followed by `regen_in_map`.
- The already-promoted `BUG-BDL-003` remains the run-completion/rejoin validation defect.

No additional source-proven lifecycle defect was promoted.

## Cursor 4 checkpoint — legacy DragonLair manager reachability
- `questlua_dragonlair.cpp` exposes `DragonLair.startRaid`, backed by `CDragonLairManager`.
- The deployed Blue Dragon quest set is `dragon_lair.quest` plus `dragon_lair_access.quest`.
- Neither deployed quest calls `DragonLair.startRaid`.
- The active entry path uses NPC 30121, global quest state and direct warp to static map 208.
- The legacy private-map manager is therefore dormant/dependency code for this pinned snapshot, not an active execution path.

## Checkpoint — 2026-09-30
Static analysis saved after lifecycle and legacy reachability closure.

Current verified Blue Dragon bugs:
- `BUG-BDL-001` — access-item loss if disconnect occurs before delayed personal entry timer.
- `BUG-BDL-002` — disconnect during locked entry dialogue can strand NPC quest lock.
- `BUG-BDL-003` — paid join remains open after Beran is already dead.
- `BUG-BDL-004` — final low-HP damage/regen factor ranges are inverted and unreachable.

Next exact step: final deployment/data parity, then close `blue_dragon_lair` if no new source-proven mismatch appears.


## Final deployment/data parity closure
- `ENABLE_BLUEDRAGON_RENEWAL` is enabled under the tracked dragon-lair feature set.
- Active `quest_list` deploys both Blue Dragon renewal quests.
- Compiled quest/object outputs exist for NPC 30121 entry/exit handlers, Beran 2493 kill, stone kills 8031-8034, login/enter/button/info hooks and both personal/server timers.
- Map 73 and map 208 data are present; tracked channel configuration places both on `ch1/core4`.
- `dragon_lair.txt` group 2430 resolves through `group.txt` to leader 2493 Beran-Setaou.
- DumpProto contains Beran 2493 and the relevant tracked mob data.
- No additional source-proven deployment mismatch was found.

## Static closure
Blue Dragon / Beran Setaou is **STATIC MAPPING CLOSED** for the pinned source snapshot.

Verified bugs:
- `BUG-BDL-001` — access-item loss on disconnect before delayed entry.
- `BUG-BDL-002` — disconnect can strand NPC quest lock.
- `BUG-BDL-003` — paid join remains open after Beran is dead.
- `BUG-BDL-004` — low-HP damage/regen factor ranges are unreachable.

Runtime execution remains locked.


## Cursor 1 checkpoint — entry authority / serialization / access transaction
- Blue Dragon map 208 is deployed only on `chan/ch1/core4` in the pinned Game snapshot. Channel 2 CONFIG files do not allow map 208.
- The DB-backed unsuffixed `dragon_lair_*` event flags therefore do not create a live cross-channel map-208 collision in this deployment; the room is effectively serialized onto the single deployed channel/core.
- Access-item and first-entry ownership paths produced `BUG-BDL-001` and `BUG-BDL-002`.
- Group-entry timing and reconnect authorization are bound to the globally stored run start time and per-player quest time.
- No additional source-proven entry/serialization defect was promoted.

### Next cursor
Boss spawn/combat hooks, Blue Dragon skill timers and stone modifiers.
