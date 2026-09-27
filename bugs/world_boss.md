# World Boss System — Bug Registry

**Status:** ACTIVE — 14 verified findings
**Phase:** Detection / Mapping Only

### BUG-WB-001 — hour/second mix-up clears spawn state and breaks scheduled cleanup
- Statik durum: **doğrulandı**
- Sınıf: scheduler / state lifecycle

At spawn, `wblast_SpawnTime` is assigned `cur_hour`.

On later one-second updates the code compares that hour value with `cur_sec` and clears `wb_Spawned` when they differ.

The intended end-of-battle cleanup branch runs only while `wb_Spawned` is true. After the spawn minute the flag is normally false, so an unkilled boss is not removed at the scheduled battle-window end.

Because `m_dwWBVID` still contains that boss VID, the next scheduled spawn also refuses to create a replacement.

### BUG-WB-002 — P2P state updates skip the originating game process
- Statik durum: **doğrulandı**
- Sınıf: P2P/local-state delivery

Spawn and kill emit only `P2P_MANAGER::Send`.

That function iterates peer descriptors and has no local loopback.

The client command fanout exists only in the receiving P2P handler. PCs attached to the process that generated the state never receive that packet through this path.

On a one-process topology the P2P peer set can be empty, so the state update reaches no player at all.

### BUG-WB-003 — ranking command is sent by the dead monster and is discarded
- Statik durum: **doğrulandı**
- Sınıf: server command routing

`CHARACTER::Reward()` finds a ranked PC as `xch`, but sends:
`ChatPacket(CHAT_TYPE_COMMAND, "worldboss_ranking ...")`

without `xch->`.

The call therefore runs on the dead monster object (`this`).

`CHARACTER::ChatPacket` returns immediately when `GetDesc()` is null. The monster has no player descriptor, so no ranking update is delivered.

### BUG-WB-004 — state command updates an unloaded throwaway window
- Statik durum: **doğrulandı**
- Sınıf: Python UI lifecycle

`game.py::__WorldbossUpdate` creates a new `uiworldboss.MainBoard()` and immediately calls `Handle()`.

It never calls `Open()`, so `__LoadScript()` has not initialized `State`, `runTime` or `breakTime`.

A normal state command reaches `AddWBState()` and accesses those missing members.

The interface module already owns a separate persistent `wndWorldBoss`; the throwaway callback object never updates it.

### BUG-WB-005 — state parser forwards list slices instead of timer/cooldown scalars
- Statik durum: **doğrulandı**
- Sınıf: client command parsing

Server command:
`update|state|timer|cooldown`

Client call:
`AddWBState(int(input[1]), input[2:], input[3:])`.

The timer becomes a two-element list and cooldown a one-element list. Rendering converts those lists to strings rather than displaying the intended scalar timestamps/countdowns.

### BUG-WB-006 — ranking parser converts player name to int
- Statik durum: **doğrulandı**
- Sınıf: client command parsing

Server sends:
`update|<playerName>|<damage>`.

Client executes:
`self.AddPlayer(int(input[1]), input[2:])`.

Normal alphabetic player names therefore raise `ValueError` before ranking row construction.

### BUG-WB-007 — ranking row renderer contains multiple independent hard failures
- Statik durum: **doğrulandı**
- Sınıf: Python UI lifecycle/rendering

Even if BUG-WB-006 is bypassed:
- `game.py` creates a fresh ranking window and calls `Handle()` without `Open()`, so header widgets are not initialized;
- `constInfo.wb_guild_names`, `wb_empire`, and `wb_tier` are lists but are invoked as functions, causing `TypeError`;
- `MakeText` overwrites the supplied text value with a `ui.TextLine` object and calls `SetText` with that object.

Thus the ranking row path has multiple successive independent failures.

### BUG-WB-008 — partial reward claim can duplicate already-granted items
- Statik durum: **doğrulandı**
- Sınıf: reward atomicity / inventory failure

Each tier reward is granted item-by-item.

If item 1 succeeds but a later item has no empty inventory position, the command returns before `SetWBRewards(true)`.

The already granted item is not rolled back.

After freeing inventory space, the normal-player `get_wb_reward` command can be called again while the reward flag remains false, granting the earlier item again.

This bug is reachable whenever the character has a nonzero World Boss tier; tier-assignment provenance remains under audit.

### BUG-WB-009 — official reward button has no bound action
- Statik durum: **doğrulandı**
- Sınıf: client UI wiring

`worldbosswindow.py` defines `reward_button`.

`uiworldboss.py::__LoadScript()` binds the page and ranking buttons but never retrieves or binds `reward_button`.

The visible official reward control therefore does nothing when clicked.


### BUG-WB-010 — ranking generation depends on item drops and misaligns player/damage iterators
- Statik durum: **doğrulandı**
- Sınıf: ranking integrity / iterator logic

Ranking construction exists only inside the branch for more than one generated drop item. A World Boss with zero or one item drop never builds ranking data.

In the ranking-enabled multi-item loop:
- `it` points to the qualifying character vector;
- `it2` points to the parallel damage vector;
- both are advanced once in the ranking block;
- `it` is then advanced a second time later in the same item iteration;
- `it2` is not.

The two parallel vectors therefore lose alignment. Damage can be attached to the wrong player, players can be skipped, and with two qualifying players one player can repeatedly occupy the ranking slot while the damage iterator alternates.

The number of ranking candidates processed is also bounded/cycled by the number of dropped items rather than by the damage participant set.

### BUG-WB-011 — timed cleanup deletes boss but leaves stale VID and dangling pointer
- Statik durum: **doğrulandı**
- Sınıf: object lifetime / scheduler state

The scheduled World Boss timeout executes:
`M2_DESTROY_CHARACTER(pkWB)`

and only sets `wb_Spawned = false`.

`M2_DESTROY_CHARACTER` calls `CHARACTER_MANAGER::DestroyCharacter`, which removes and deletes the character but does not invoke World Boss `OnKill()`.

Consequently:
- `m_dwWBVID` remains nonzero;
- `pkWB` still points to the deleted object;
- World Boss phase/cooldown state is not transitioned.

The next spawn path requires `m_dwWBVID == 0`, so scheduled spawning can remain permanently blocked. The stale `pkWB` is also a dangling pointer.

This defect is independent of BUG-WB-001: BUG-WB-001 usually prevents the timeout branch; BUG-WB-011 describes the broken state transition if that branch does execute.

### BUG-WB-012 — disabling the event leaves active/stale World Boss state unmanaged
- Statik durum: **doğrulandı**
- Sınıf: event lifecycle / stale manager state

`CEventManager::SetWorldBossEvent(false)` only updates the event flag and broadcasts an event-completed notice.

It does not destroy an active World Boss and does not clear manager state.

World Boss scheduler work is gated by `world_boss_event == 1`, so timeout management stops while the event is disabled.

The World Boss death hook is also gated by the same event flag. If the boss dies after the event has been disabled, `CHARACTER_MANAGER::OnKill` is not called and the stored World Boss VID/pointer/phase state remains stale.

A later event activation can therefore inherit an unmanaged existing boss or stale nonzero `m_dwWBVID` that prevents a fresh spawn.


### BUG-WB-013 — players joining after a transition cannot synchronize current World Boss state
- Statik durum: **doğrulandı**
- Sınıf: session synchronization / transient state delivery

World Boss state reaches clients only through the command emitted by `CInputP2P::WorldBoss` when a peer state packet is received.

No World Boss state send exists in the mapped login/input initialization paths.

The persistent client World Boss window does not request state when opened; `Open()` only loads the UI script and shows the window.

Consequently a player logging in or reconnecting after the last spawn/kill transition receives no current state, boss timer, or cooldown value. The window can remain at local/default/stale values until another server transition command happens.

This also compounds BUG-WB-002 for players on the originating process.


### BUG-WB-014 — World Boss ownership is process-local, allowing duplicate simultaneous bosses across cores
- Statik durum: **doğrulandı**
- Sınıf: multi-core ownership / scheduler coordination

Each game process owns an independent `CHARACTER_MANAGER` and independent World Boss state (`m_dwWBVID`, `pkWB`, phase/state/cooldown), plus process-local `wb_Spawned` / `wblast_SpawnTime`.

Every process runs the World Boss scheduler itself and randomly selects a candidate map. `map_allow_find` only filters whether that selected map is hosted by the local process.

The World Boss P2P receive path does not copy remote ownership state into the local manager and provides no election, lock, or "boss already exists elsewhere" guard; it only forwards the received state to local PCs.

Consequently, in a multi-core/channel topology where different game processes host eligible World Boss maps, two or more processes can independently pass their local `m_dwWBVID == 0` check and spawn separate World Boss instances during the same scheduled window.
