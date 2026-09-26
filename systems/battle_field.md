# Battle Field System

**Status:** PARTIAL — ACTIVE
**Phase:** Detection / Mapping Only
**Date:** 2026-09-27

> Source repositories are read-only. This map records static architecture/findings only.

## Initial source roots
Server:
- `game/src/battle_field.h`
- `game/src/battle_field.cpp`
- `game/src/ranking_system.*` only where Battle Field ranking integration crosses subsystem boundaries
- `game/src/cmd_general.cpp` Battle Field commands
- `game/src/input_p2p.cpp` Battle Field command propagation
- `game/src/packet.h` Battle Field packets
- quest/event flags and DB tables used by Battle Field

Client:
- `root/uibattlefield.py`
- `root/interfacemodule.py::OpenRankingBoardWindow`
- `root/game.py` Battle Field command callbacks
- `UserInterface/PythonNetworkStreamPhaseGame.cpp::RecvBattleZoneInfo`
- player/network Battle Field bindings

## Known architecture from completed Ranking cross-audit
- Battle Field ranking cache is loaded by `CRankingSystem`.
- `CBattleField::OpenBattleUI` sends field state/timing and ranking data.
- ranking GC transport uses dynamic `HEADER_GC_BATTLE_ZONE_INFO`.
- client clears/rebuilds ranking cache then opens Battle Field UI.
- Battle Field UI has current/accumulated ranking tabs.

Ranking-specific defects remain canonical under `bugs/ranking.md`; they are not duplicated here unless a distinct Battle Field lifecycle bug is found.

## Battle Field state surface
Already observed server concepts:
- open/closed/disabled status via quest event flag;
- configured open/close schedule loaded from SQL;
- ranking update schedule loaded from SQL;
- enter/exit cooldown;
- event-mode score multiplier;
- temporary in-session Battle Field points;
- persistent `POINT_BATTLE_FIELD`;
- player kill/death handling;
- random respawn positions;
- weekly ranking rollover and winner affects;
- P2P broadcast commands for UI/event state.

## Exact next audit
1. Map entry/exit command authorization and map/channel restrictions.
2. Map kill/death/score accounting and persistence.
3. Audit schedule calculations for day/time boundary errors.
4. Audit cooldown and reconnect/disconnect behavior.
5. Audit P2P state synchronization across cores.
6. Start Battle Field-specific bug registry; do not duplicate Ranking bugs.


## Entry / exit command audit — PARTIAL CLOSED

Player command registrations:
- `open_battle_ui` -> GM_PLAYER / POS_DEAD
- `goto_battle` -> GM_PLAYER / POS_DEAD
- `exit_battle_field` -> GM_PLAYER / POS_DEAD
- `exit_battle_field_on_dead` -> GM_PLAYER / POS_DEAD
- Battle-specific `restart_immediate` -> GM_PLAYER / POS_DEAD.

`goto_battle -> RequestEnter` has server-side guards for:
- active/open status;
- minimum level 50;
- not channel 99;
- not a private map;
- not riding;
- `CanWarp()`;
- 600-second return cooldown.

The Battle-specific restart branch separately verifies the player is actually on the Battle Field map and has the required item, so it was rejected as a false-positive command-boundary bug.

Exit paths are weaker:
- `RequestExit` has no Battle Field map-membership check -> BUG-BFIELD-001.
- `exit_battle_field_on_dead 1` bypasses map, death, `CanWarp` and exit-flow checks -> BUG-BFIELD-002.

## Kill-score anti-farming audit
`RewardKiller` correctly validates both participants are PCs on the Battle Field map, requires descriptors, rejects equal host names and calls `SetBattleKill(victimPID)`.

Configured repeat interval:
`BATTLE_FIELD_KILL_TIME = 60` seconds.

`SetBattleKill` does not replace an expired map entry; it calls `emplace` on an already-existing PID. After the first expiry, the timestamp stays permanently in the past for that victim during the character session -> BUG-BFIELD-003.

Temporary Battle Field points are explicitly initialized to zero in character initialization, along with the kill map and death-limit counter.

## Current verified Battle Field bugs
- `BUG-BFIELD-001`
- `BUG-BFIELD-002`
- `BUG-BFIELD-003`

## Exact next audit
1. Finish schedule/open-close resolver correctness.
2. Audit event-mode state and P2P synchronization across cores.
3. Trace Battle Field connect/disconnect/party removal into existing Party invariants.
4. Audit score cash-out/ranking update atomicity and disconnect-loss behavior.
5. Audit daily shop-point reset state.


## Ranking integration / rollover audit

### BUG-BFIELD-004 — unresolved LoadRanking call
The Battle Field source calls unqualified `LoadRanking(RK_CATEGORY_BF)` in:
- `CloseEnter`;
- scheduled weekly update inside `Update`.

The current server build defines both feature macros, but the only mapped API is `CRankingSystem::LoadRanking`. No CBattleField/global wrapper is declared in the mapped include chain.

### BUG-BFIELD-005 — stale weekly winner slots
`UpdateWeekRanking` overwrites only winner positions that exist in the new top-3 result and does not clear unused `battle_week` rows.

`LoadRankingWeekWinners` has no current-week filter, so stale rows survive into the winner cache whenever a rollover has fewer than three qualifying players.

## Cross-system Party reachability
Battle Field `Connect` removes an entering character from its party via:
`party->Quit(playerID)`.

The Party registry already records this as normal reachability for:
- `BUG-PARTY-001` leader self-delete/use-after-free;
- `BUG-PARTY-005` stale party role bonuses.

No duplicate Battle Field bug ID is assigned for those underlying Party defects.
