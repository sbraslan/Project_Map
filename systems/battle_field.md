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


## Event-state / multi-core audit

### BUG-BFIELD-006 — event date is process-local and battle_set_event is not channel-gated
`do_battle_set_event` writes only `CBattleField::bEventMonth/bEventDay`.

Force-open/force-close explicitly reject non-Battle-Field channels, but `battle_set_event` does not.

The main heartbeat invokes `CBattleField::Update()` only on channel 99. Thus setting the event date from another channel modifies an inert local singleton instead of the scheduler-owning core.

P2P `BroadcastCommand` paths transport client commands; they do not synchronize these server singleton fields.

## Weekly winner affect lifecycle

### BUG-BFIELD-007 — online players keep stale winner flags across rollover
Winner flags are written directly into `m_afAffectFlag` by `SetWeakRankingPosition`.

The weekly DB/cache reload does not iterate online characters, reset old `AFF_BATTLE_RANKER_1..3`, or assign flags to newly ranked online characters.

This is separate from BUG-BFIELD-005:
- 005 concerns stale DB rows when fewer than three winners exist;
- 007 exists even with a perfectly correct fresh top three because online character flags are not reconciled.

## Schedule resolver audit

### BUG-BFIELD-008 — second component has wrong sign
Same-day remaining time adds current `tm_sec` instead of subtracting it, creating a 0..118 second overstatement.

Next-day calculation has a -60..58 second error for the same reason combined with the minute rollover formula.

The schedule fallback only searches today and the immediate next day. Sparse schedules can therefore resolve to zero when the next configured opening is two or more days away, but the live SQL schedule rows are absent from this repository snapshot, so this remains an unpromoted config-dependent candidate.

## Battle shop daily state — CLOSED
Battle shop usable-point and last-reset timestamps live in `TPlayerTable::aiShopExUsablePoint/aiShopExDailyUse`.

They are copied into player-save data and the Battle shop purchase path triggers character save after spending.

`Connect` restores the allowance when more than 86400 seconds have elapsed since the saved timestamp.

This is a rolling 24-hour allowance rather than a calendar-day reset; no separate persistence failure was established from the mapped path.

## Current verified Battle Field set
- BUG-BFIELD-001
- BUG-BFIELD-002
- BUG-BFIELD-003
- BUG-BFIELD-004
- BUG-BFIELD-005
- BUG-BFIELD-006
- BUG-BFIELD-007
- BUG-BFIELD-008

## Remaining before static close
1. Audit disconnect/reconnect behavior for temporary score and cooldown semantics.
2. Finish client event-state command wiring; classify the unused/mismatched event-enable fields.
3. Audit close/open state propagation to connected clients across cores.
4. Recheck score cash-out -> DB ranking write ordering.
5. Decide Battle Field STATIC COMPLETE.


## Score cash-out / persistence boundary

### BUG-BFIELD-009 — cap rejection is ignored
The exit sequence clears temporary score and writes ranking after calling the void-returning persistent Battle Point change.

If the persistent balance + temporary score reaches/exceeds `BATTLE_POINT_MAX`, `PointChange` refuses the currency update, while the temporary score is still zeroed and ranking is still credited.

`WarpSet` later calls `Save()`, so the normal successful path schedules player persistence after the Battle Point mutation. The ranking write itself is a separate direct SQL operation, so the overall currency/ranking update is not a single DB transaction; crash consistency remains a separate fault-injection concern and is not promoted statically.

## Client event-state wiring

### BUG-BFIELD-010 — generic open propagates, event-open does not
Client command registry exposes both `battle_field_event` and `battle_field_event_open`.

`OpenEnter(isEvent)` only broadcasts `battle_field_open 1` before setting local event status. It never broadcasts event=true/event-open=true.

Normal channel `Connect` cannot repair this because its local Battle Field singleton has `bEventStatus=false`.

The event-specific minimap branch therefore lacks the state transition it expects.

Latent/non-promoted:
- `GetBattleFieldEventEnable()` returns the event-open field instead of event-enable.
- current minimap stores that getter result but does not use it in branching.

## Disconnect/reconnect semantics — no new verified bug
Temporary `dwBattleFieldPoints` is in-memory only and reset to zero on character initialization. `CHARACTER::Disconnect` does not execute `ExitCharacter`, so disconnecting inside Battle Field does not cash temporary points or set the 600-second exit cooldown.

This creates a clear forfeit/bypass semantic, but the source does not establish whether disconnect is intentionally defined as forfeiting unbanked temporary score. It remains documented rather than promoted.

## Current verified Battle Field set
- BUG-BFIELD-001
- BUG-BFIELD-002
- BUG-BFIELD-003
- BUG-BFIELD-004
- BUG-BFIELD-005
- BUG-BFIELD-006
- BUG-BFIELD-007
- BUG-BFIELD-008
- BUG-BFIELD-009
- BUG-BFIELD-010

## Remaining before static close
1. Recheck Battle Field open/close transition behavior for sparse multi-day schedules; keep unpromoted if DB reachability remains unknown.
2. Verify daily-reset initialization/load path once more.
3. Verify Battle Field death-limit use/reset consumer.
4. Close remaining client command/state surfaces.
5. Decide Battle Field STATIC COMPLETE.
