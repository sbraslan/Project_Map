# Blue Dragon / Beran Setaou — Static System Map

**Status:** STATIC MAPPING OPEN / 3 VERIFIED BUGS
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
