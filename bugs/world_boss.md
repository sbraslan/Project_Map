# World Boss System — Bug Registry

**Status:** ACTIVE — 9 verified findings
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
