# World Boss System

**Status:** STATIC COMPLETE
**Phase:** Detection / Mapping Only
**Date:** 2026-09-27

> Source/game repositories are read-only. Only Project_Map is writable.

## Feature/config
`ENABLE_WORLD_BOSS` and `ENABLE_WB_RANKING` are enabled.

Configured constants:
- `WORLD_BOSS_PHASE = 6`
- `BATTLE_PHASE = 4`
- `COOLDOWN_PHASE = 2`
- `WB_MIN_DMG = 60000`

World Boss event activation is owned by `CEventManager::SetWorldBossEvent()` through quest flag `world_boss_event`.

## Server roots
- `game/src/char_manager.cpp/.h` — spawn/despawn state, World Boss VID, map/vnum lists, P2P state emission.
- `game/src/char_battle.cpp` — World Boss death hook and damage/ranking generation.
- `game/src/input_p2p.cpp` — received World Boss P2P state -> client command fanout.
- `game/src/p2p.cpp/.h` — peer-only P2P transmission.
- `game/src/packet.h` — `HEADER_GG_WORLD_BOSS` / `TPacketGGSendWorldBossStates`.
- `game/src/cmd_general.cpp::do_get_wb_reward` — tier reward claim.
- `game/src/cmd.cpp` — `get_wb_reward` registered for `GM_PLAYER`.
- `game/src/char.h` — per-character tier and reward flag.
- `game/src/char.cpp::ChatPacket` — descriptor-gated command delivery.
- `game/src/event_manager.cpp` — event activation flag.

## Client roots
- `root/game.py` — `worldboss` and `worldboss_ranking` server-command callbacks.
- `root/interfacemodule.py` — persistent World Boss / ranking windows.
- `root/uiworldboss.py`
- `root/uiworldbossranking.py`
- `root/constinfo.py` — global ranking arrays.
- `root/uiscript/worldbosswindow.py`
- `root/uiscript/worldbossrankingwindow.py`

## Spawn / state lifecycle
World Boss candidate maps are 61, 62, 63 and 64. Current vnum list contains five entries, all 1093.

Each game process checks the active event once per second. At configured spawn hours it can spawn a boss on an allowed randomly selected World Boss map.

After spawn:
- `m_dwWBVID` stores the boss VID;
- `m_lWBPhase` is set to now + 4 hours;
- state becomes `WORLD_BOSS_STATE_BOSS_SPAWNED`;
- a `HEADER_GG_WORLD_BOSS` packet is sent to P2P peers.

### Spawn-state timer defect
`wblast_SpawnTime` is assigned `cur_hour` at spawn.

On later updates the code compares that stored hour against `cur_sec`:
`if (wblast_SpawnTime != cur_sec) wb_Spawned = false;`

This normally clears `wb_Spawned` immediately after the spawn minute. The scheduled 4-hour cleanup branch requires `wb_Spawned == true`; therefore an unkilled boss can survive past its intended battle window, and `m_dwWBVID != 0` then blocks the next scheduled spawn. See BUG-WB-001.

## P2P state delivery
Spawn and kill paths call only:
`P2P_MANAGER::Instance().Send(&pack, sizeof(pack))`.

`P2P_MANAGER::Send` iterates `m_set_pkPeers` and writes the packet only to peer descriptors. It has no local loopback.

Only the receiving peer's `CInputP2P::WorldBoss` fans state out to that process's PCs using:
`worldboss update|state|timer|cooldown`.

Therefore PCs attached to the game process that originated the spawn/kill state do not receive that state command. On a one-process setup, no PC receives it at all. See BUG-WB-002.

## Server ranking generation
World Boss ranking data is built in `CHARACTER::Reward()` from damage ownership data.

The server formats:
`worldboss_ranking update|<playerName>|<damage>`.

However the call is unqualified:
`ChatPacket(...)`

inside the dead monster's member function. It therefore executes on `this` (the World Boss), not on `xch`.

`CHARACTER::ChatPacket` immediately returns when `GetDesc()` is null. Monsters do not own a player descriptor, so the ranking command is discarded. See BUG-WB-003.

## Client state command path
`root/game.py` registers:
- `worldboss` -> `__WorldbossUpdate`
- `worldboss_ranking` -> `__WorldbossRanking`.

The interface module already owns persistent windows:
- `self.wndWorldBoss`
- `self.wndWBRanking`.

But each server-command callback creates a new temporary window instance instead of updating those interface windows.

For state updates, `MainBoard()` is constructed and `Handle()` is called without `Open()` / `__LoadScript()`. Widget members such as `self.State`, `self.runTime`, and `self.breakTime` are assigned only by `__LoadScript()`. A normal state command therefore reaches `AddWBState()` with missing widgets and can raise `AttributeError`; it also never updates the visible persistent window. See BUG-WB-004.

The state parser also receives:
`update|state|timer|cooldown`

but forwards:
`input[2:]` and `input[3:]`

instead of scalar `input[2]` and `input[3]`. Even after the unloaded-window defect is corrected, timer/cooldown become list values and render as list-string representations. See BUG-WB-005.

## Client ranking path
The server format is:
`update|<playerName>|<damage>`.

The client immediately executes:
`int(input[1])`.

Normal alphabetic character names therefore raise `ValueError` before a row can be created. See BUG-WB-006.

Additional independent ranking renderer defects remain behind that first failure:
- `wb_guild_names`, `wb_empire`, and `wb_tier` are lists in `constInfo`, but `AddPlayer()` calls them as functions, causing `TypeError`.
- `MakeText(parent, text, ...)` immediately replaces its `text` argument with a `ui.TextLine` object and then calls `SetText(text)`, discarding the supplied row value.
- `game.py::__WorldbossRanking` also creates a fresh window and calls `Handle()` without `Open()`, so ranking headers are not initialized and the persistent interface ranking window is not updated.

These independent rendering failures are grouped under BUG-WB-007 because they are successive failures in the same row-construction path.

The global ranking arrays and `WB_RANKS` counter are not reset by `Open()`, `__LoadScript()`, or destruction. If the upstream ranking path is repaired, later boss results can accumulate stale prior rows. This remains a mapped latent cache issue pending ranking-lifecycle closure.

## Reward claim
`get_wb_reward` is registered as a normal-player command (`GM_PLAYER`).

The command blocks:
- observer mode;
- dead/stunned characters;
- already-rewarded characters;
- tier 0.

Each tier currently contains three reward item vnums: 19, 29 and 39.

### Partial-grant duplication window
Items are created and granted one by one. If an early item is successfully granted but a later item finds no empty inventory slot, the function returns immediately.

`SetWBRewards(true)` runs only after the whole item loop succeeds.

Thus a character with just enough space for an early item can receive it, fail on a later item, keep `GotWBRewards() == false`, free space and call the command again to receive the early item again. See BUG-WB-008.

## Client reward button
The UI script defines `reward_button`, but `uiworldboss.py::__LoadScript()` never retrieves it and never binds an event to `get_wb_reward`.

The official World Boss window therefore exposes a visible reward button with no functional click handler. See BUG-WB-009.

## Verified bugs
- `BUG-WB-001` — spawn hour is stored but compared to current second, clearing `wb_Spawned` and breaking timed cleanup / future spawn progression for an unkilled boss.
- `BUG-WB-002` — World Boss P2P state emission has no local loopback; players on the originating game process miss spawn/kill state updates.
- `BUG-WB-003` — ranking command is sent through the dead boss's `ChatPacket`, whose null descriptor drops the command.
- `BUG-WB-004` — state callback creates an unloaded throwaway UI window, causing missing-widget failure and leaving the persistent World Boss window stale.
- `BUG-WB-005` — state parser passes timer/cooldown slices instead of scalar fields.
- `BUG-WB-006` — ranking parser casts normal player names to `int`, causing `ValueError`.
- `BUG-WB-007` — ranking row construction is independently broken by list-as-function calls, unloaded temporary window usage, and text-argument destruction.
- `BUG-WB-008` — reward flag is set only after all items; mid-bundle inventory failure permits repeated partial reward claims.
- `BUG-WB-009` — official reward button is never bound to any handler/command.
- `BUG-WB-010` — ranking generation is coupled to multi-item drops and advances character/damage iterators out of sync.
- `BUG-WB-011` — timed boss destruction deletes the object without clearing World Boss VID/pointer state.
- `BUG-WB-012` — disabling the event does not tear down the active boss/state, and deaths while disabled bypass `OnKill()`.
- `BUG-WB-013` — login/reconnect has no current-state synchronization; players joining mid-cycle can remain permanently stale until another transition.

## Ranking ownership-loop closure
World Boss ranking collection is embedded inside the ordinary multi-item drop ownership loop rather than being derived independently from the damage map.

It runs only when:
- `CreateDropItem(...)` succeeds;
- the drop vector contains more than one item;
- at least one damage owner meets the 10% ownership threshold.

Therefore a World Boss producing zero or one item generates no ranking rows at all, regardless of player damage.

Inside the ranking-enabled multi-item loop, the character iterator is advanced once in the ranking block and then again later in the same item iteration, while the damage iterator is advanced only once. Character and damage vectors therefore become misaligned. With two qualifying players, the same character can be selected repeatedly while damage values alternate; with three or more players, damage can be attributed to the wrong character and some players are skipped.

See BUG-WB-010.

## Timed cleanup state corruption
The scheduled timeout path calls:
`M2_DESTROY_CHARACTER(pkWB)`

and then only clears `wb_Spawned`.

`M2_DESTROY_CHARACTER` maps to `CHARACTER_MANAGER::DestroyCharacter`. That function removes/deletes the character but does not invoke World Boss `OnKill()` and does not clear:
- `m_dwWBVID`;
- `pkWB`;
- World Boss phase/cooldown state.

So even if BUG-WB-001 were corrected and timeout cleanup executed, the boss object would be deleted while `pkWB` remains a dangling pointer and `m_dwWBVID` remains nonzero. The next scheduled spawn is then blocked by `m_dwWBVID == 0`. See BUG-WB-011.

## Event-disable lifecycle
`CEventManager::SetWorldBossEvent(false)` only changes the `world_boss_event` flag and sends a completion notice.

It does not destroy an active World Boss or reset manager state.

The World Boss death hook in `char_battle.cpp` is itself conditional on `world_boss_event == 1`. If a boss remains alive when the event is disabled and then dies while disabled, `OnKill()` is not called, leaving stale World Boss VID/pointer/state. If it simply remains alive, the disabled scheduler no longer manages its timeout.

See BUG-WB-012.

## Login / reconnect state synchronization
The World Boss client has no request/response path for current state.

Mapped login/input paths contain no World Boss state sync, and opening `wndWorldBoss` only loads/shows the local Python UI; it does not send a network request.

The only mapped server-to-client state command is produced by the P2P receive handler when it receives a spawn/kill state packet.

Therefore a player who logs in after the last spawn/kill transition, reconnects mid-cycle, or otherwise misses that transient command has no way to reconstruct current boss state/timers until a later transition occurs. See BUG-WB-013.

## Tier/reward persistence note
`m_pTier` and `m_pGotRewards` are plain CHARACTER members initialized on character construction to 0 / false.

No corresponding fields exist in the mapped `TPlayerTable` or player DB load/save path.

Thus the state is session-local rather than persisted. Concrete user-facing severity depends on where/when `SetTier()` is called; tier-assignment provenance remains open, so no separate bug ID is assigned yet.

## Open audit
1. Finish tier-assignment provenance: identify whether `SetTier()` has any live caller.
2. Audit reward-state reset semantics across successive boss cycles.
3. Audit multi-core/channel ownership to determine whether multiple simultaneous bosses are intended or accidental.
4. Audit ranking cache reset/pagination behavior after upstream routing/parser defects.


## Multi-core/channel ownership closure
World Boss scheduler/ownership state is process-local:
- `m_dwWBVID`, `pkWB`, `m_lWBPhase`, `m_lWBCooldown`, and `m_bWBState` are members of each process-local `CHARACTER_MANAGER`;
- `wb_Spawned` and `wblast_SpawnTime` are process-local globals in `char_manager.cpp`;
- every game process independently runs the scheduler and independently selects a random entry from `WBMapIndexes`;
- `map_allow_find(WB_MAP_INDEX)` only decides whether that process hosts the randomly selected map.

The World Boss P2P receive handler does not update any manager ownership field. It only broadcasts a client command to PCs attached to the receiving process.

Therefore there is no cross-core election/lock or authoritative World Boss owner. If multiple game processes host eligible World Boss maps, more than one process can independently satisfy its spawn condition and create a boss during the same scheduled spawn window. Each process then tracks only its own local VID/pointer.

See BUG-WB-014.

## Tier / reward reset audit status
The mapped World Boss roots still show `m_pTier = 0` and `m_pGotRewards = false` only at CHARACTER construction. The live reward command reads `GetTier()` and sets `SetWBRewards(true)` after a successful bundle.

No tier assignment or per-boss `SetWBRewards(false)` reset exists in the mapped World Boss lifecycle paths (spawn, death/ranking, P2P state, event toggle, reward command). Repository-wide code-search indexing is unavailable, so caller provenance is kept open rather than promoted to a verified bug without a complete negative proof.

## Ranking cache/pagination audit status
The official Python assets are located in `Project_Binary/root` (not Project_Game):
- `constinfo.py`
- `uiworldbossranking.py`
- `game.py`
- `interfacemodule.py`
- `uiscript/worldbossrankingwindow.py`

`constInfo.WB_RANKS` and the parallel `wb_*` arrays are initialized only at module load. `uiworldbossranking.MainWindow.Open()` loads/shows the window but does not clear them, and the mapped interface teardown destroys the window without resetting the module globals.

There is also no mapped page/scroll/reset protocol for World Boss ranking rows. Because the current server-routing/parser defects prevent the normal ranking path from functioning, this remains a latent lifecycle/cache defect rather than a new verified user-facing bug ID in the current build.


## Damage-owner lifetime / disconnect audit
At death/reward time the ordinary damage map is converted back to live character pointers with:
`CHARACTER_MANAGER::Instance().Find(it->first)`.

A damage entry contributes to the priority queue and to `total_dam` only when that VID still resolves to a live character on the local process.

Therefore a participant who disconnects or otherwise ceases to resolve locally before boss death is omitted entirely; that participant's accumulated damage is also removed from the denominator used by the 10% ownership threshold. This can change which remaining players qualify for ownership/ranking compared with the actual fight damage history.

The behavior is now mapped, but no bug ID is assigned yet because the source does not establish whether "must still be locally present at death" is intentional eligibility policy or an unintended World Boss ranking rule.


## Scheduler clock / event propagation closure
The scheduler derives its hour from:
`gmtime(system_clock::now())`
followed by:
`cur_hour = utc_tm.tm_hour + 2`.

Consequences:
- the value is not normalized modulo 24, so late UTC hours produce 24 and 25;
- the spawn condition explicitly includes `cur_hour == 24` and `cur_hour == 0`, so the midnight spawn branch is intentionally compensating for one overflow case;
- the fixed `+2` offset has no daylight-saving/timezone rule and therefore represents a fixed UTC+2 schedule, not a timezone-aware local clock.

No new verified bug ID is assigned from this alone because the source does not establish whether the intended production schedule is fixed UTC+2 or civil local time. If production expects Europe/Berlin-style local time, winter schedules will shift by one hour.

Event activation propagation itself is not the multi-core defect:
- `CEventManager::UpdateGameFlag()` immediately updates the originating core's quest flag;
- it sends `HEADER_GD_EVENT_NOTIFICATION` to DB;
- DB `CClientManager::EventNotification()` forwards `HEADER_DG_EVENT_NOTIFICATION` to connected game peers;
- each receiving game process calls `CQuestManager::SetEventFlag()`.

Therefore `world_boss_event` is distributed across game processes. BUG-WB-014 remains specifically an ownership/scheduler coordination problem, not a missing event-flag broadcast.

### Same-minute retry behavior
The scheduler executes once per second while `cur_min == 0`.
A successful spawn sets `m_dwWBVID`, which blocks another successful spawn on that process during the same minute. If spawning fails and `m_dwWBVID` remains zero, the code can retry on later scheduler ticks in the same minute. This is mapped behavior, not presently classified as a bug.


## Death / reward / manager-state ordering
The World Boss death path in `CHARACTER::Dead()` processes ordinary monster reward logic before the World Boss manager callback.

Relevant order:
1. monster death state is entered;
2. if rewards are allowed, `Reward(true)` executes and builds item ownership / World Boss ranking data;
3. later in the same `Dead()` function, the `ENABLE_WORLD_BOSS` block checks the event flag and race vnum;
4. only then does `CHARACTER_MANAGER::OnKill(GetVID())` clear `m_dwWBVID/pkWB` and publish killed/cooldown state.

No new race/lifetime defect was found from this ordering itself: the boss object and damage map are still available while `Reward()` runs, and manager ownership is cleared afterwards.

This audit also confirms that no World Boss tier assignment or reward-eligibility reset is performed in the death callback path itself. The tier/reward provenance question therefore remains isolated outside the mapped spawn/death/reward lifecycle.


## Client titlebar parent/child visibility mismatch
Both World Boss UI scripts use a `board_with_titlebar` child inside a parent `ui.ScriptWindow`:
- `uiworldboss.MainBoard`
- `uiworldbossranking.MainWindow`

Neither class rebinds the board close event to the parent window and neither defines a dedicated `Close()` / Escape handler.

Framework behavior in `ui.BoardWithTitleBar.__init__()` is:
`self.SetCloseEvent(self.Hide)`

That callback belongs to the BoardWithTitleBar child itself, so clicking the titlebar X hides only the board child, not the owning ScriptWindow.

The interface toggles, however, test `wndWorldBoss.IsShow()` / `wndWBRanking.IsShow()` on the parent ScriptWindow. This creates a parent/child visibility mismatch: after clicking X, the visible content disappears while the parent can remain logically shown. The next toggle action therefore hides the already-invisible parent; only a subsequent toggle reopens/reloads it.

See BUG-WB-015.

## External quest/data root audit
The current World Boss vnum table contains vnum 1093. Project_Game contains existing `1093.kill` quest handlers:
- Devil Tower 1093 handler is guarded by dungeon/map 660000-669999 and therefore does not trigger on World Boss maps 61-64.
- Biolog level-90 quest intentionally lists 1093 among several valid kill targets and can grant its normal quest drop when a player in the relevant quest state kills the World Boss.

No World Boss-specific tier assignment or reward reset was found in the named quest/data roots. The biolog overlap is documented as integration behavior, not classified as a bug because the quest intentionally treats vnum 1093 as a general eligible target.


## Reward command / tier provenance closure
`get_wb_reward` is registered at `GM_PLAYER`, so ordinary players may invoke it through the command interpreter.

The command gates on:
- not observer;
- not dead/stunned;
- `GotWBRewards() == false`;
- `GetTier() != 0`.

However the mapped runtime provenance for tier/reward state is incomplete in the implementation itself:
- `m_pTier` and `m_pGotRewards` are private CHARACTER members;
- `CHARACTER::Initialize()` sets `m_pTier = 0` and `m_pGotRewards = false`;
- neither value exists in `TPlayerTable`, so login/save does not persist or restore them;
- the quest Lua bindings audited (`questlua_pc.cpp`, `questlua_game.cpp`, `questlua_global.cpp`) expose no World Boss tier setter;
- login/main input paths expose no World Boss tier packet;
- World Boss spawn, damage/ranking, death, P2P-state and event-manager paths do not assign a tier;
- the reward command itself only reads `GetTier()` and finally sets `SetWBRewards(true)`.

Therefore, in the mapped current build, a normal player starts each CHARACTER session at tier 0 and no connected World Boss lifecycle path promotes that value above zero. The reward command consequently exits before granting any reward.

See BUG-WB-016.

### Reward flag lifetime
`m_pGotRewards` is also session-local and not persistent. A reconnect creates a new CHARACTER and resets it to false. This does not currently create a repeat-claim exploit by itself because tier also resets to zero and has no mapped reassignment path. If a future tier assignment is added without event-scoped/persistent claim state, reconnect semantics must be retested.


## Static closure
World Boss static mapping is complete for the current source snapshots.

Closed roots include:
- scheduler/spawn/despawn;
- event flag propagation;
- process-local multi-core ownership;
- P2P state transport and local delivery;
- login/reconnect synchronization;
- death/reward ordering;
- damage/ranking ownership;
- reward command registration and eligibility;
- session/persistence provenance for tier/reward state;
- client command parsing and UI lifecycle;
- ranking cache/window lifecycle;
- titlebar close behavior;
- World Boss vnum integration with current quest data.

Verified registry: `BUG-WB-001..016`.

Runtime/fault-injection validation remains deferred to `tests/world_boss.md`; source/game repositories remain unchanged.
