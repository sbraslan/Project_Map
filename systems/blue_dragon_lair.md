# Blue Dragon / Beran Setaou — Static System Map

**Status:** STATIC MAPPING OPEN / 2 VERIFIED BUGS
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
